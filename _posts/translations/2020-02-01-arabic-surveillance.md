---
layout: post
title: Массовая слежка от ОАЭ. Исследование очень популярного приложения для iOS с тёмной стороной
category: [translations]
tag: [ios, reverse-engineering]
---

[Ссылка на оригинал](https://objective-see.com/blog/blog_0x52.html)

Спасибо большое за ревью перевода @Franky_T!

## Intro

Хочешь проверить приложение сам?

Я загрузил расшифрованную копию приложения и расшифровку перехватов трафика:

Скачать: [ToTok.zip](https://objective-see.com/downloads/blog/blog_0x52/ToTok.zip), [Traffic.zip](https://objective-see.com/downloads/blog/blog_0x52/Traffic.zip)

*Прим.пер. - файлы перезалиты на мой сайт - [ToTok.zip](/assets/files/ToTok.zip), [Traffic.zip](/assets/files/Traffic.zip)*

## Как все начиналось

Недавно ко мне обратилась газета _New York Times_ с просьбой помочь в расследовании, связанном с популярным приложением для iOS - _ToTok_.

Видимо, "американские чиновники, знакомые с секретной разведкой" установили, что ToTok, на самом деле, является шпионским инструментом. 

> _"Он используется правительством Объединённых Арабских Эмиратов, чтобы попытаться отследить каждый разговор, движение, взаимоотношения, встречи, аудио данные и изображения тех, кто устанавливает его на свои телефоны". - New York Times_

Отчет _New York Times_: [It Seemed Like a Popular Chat App. It’s Secretly a Spy Tool.](https://www.nytimes.com/2019/12/22/us/politics/totok-app-uae.html)

![](/assets/images/ru/ios/1.png)

Сегодня мы исследуем iOS приложение _ToTok_.

> Основная цель этого поста - рассказать о технической стороне исследования iOS приложений, на примере _ToTok_.
> Проще говоря, попытаемся обсудить процесс бинарного анализа и отметить несколько интересных наблюдений.

_ToTok_ (разработчик "Breej Holding Ltd.") является очень популярным приложением в Объединенных Арабских Эмиратах (ОАЭ). По факту, недавно это было "трендовое" приложение №1 в Дубае:

![](/assets/images/ru/ios/2.png)

... тем временем, в магазине приложений для iOS рейтинг этого приложения тоже довольно высок: 

![](/assets/images/ru/ios/3.png)

Отзывы (более 32 000!) во многом положительные, и в основном хвалят тот факт, что это приложение не блокируется в ОАЭ (Skype, WhatsApp и т.д. блокируются, в то время как использование VPN для доступа к заблокированным сервисам является незаконным).

> _"Наконец-то VoIP-приложение, которое работает в ОАЭ. К счастью, оно таким было с самого начала. Четкость голоса и видео просто потрясающая!!! Большое спасибо ToTol и TRA" - Mustafa Abdul Ahad_

> _"Правда, спасибо за это приложение. Я могу, наконец, позвонить или сделать видео звонок со своей семьей, в то время, как другие приложения запрещены в моей текущей стране проживания. Спасибо" - Jckarhmrv_

> _"Я никогда не оставлял положительных комментариев или отзывов, но TOTOK заставил меня оценить ваши усилия и подарить любовь вашему непревзойденному приложению. В ОАЭ ТОТОК похож на воду 

> _"...Я хочу выразить огромную благодарность всей вашей команде за то, что вы, ребята, проделали отличную работу 

> _"Лучшее приложение 2019 года, впечатляющая разработка. Самое главное, что оно работает в ОАЭ, и это настоящее благословение" - BeingGaurav_

> _"Здорово иметь такое приложение в стране, где запрещены аудио и видео звонки!" -baghya_

... возникает ощущение, что ToTok слишком хорош, чтобы быть правдой! 

Это приложение также рекомендуется на различных сайтах в интернете, особенно в качестве замены других приложений, блокируемых в ОАЭ: 

![](/assets/images/ru/ios/4.png)

Похоже, что приложение было [удалено](https://www.thenational.ae/uae/uae-residents-confused-by-removal-of-totok-from-apple-store-1.954596) из iOS App Store!

![](/assets/images/ru/ios/5.png)

## Подготовка к анализу

Анализ iOS приложений - не самый простой процесс, так как они распространяются (через App Store) в зашифрованном виде.

Существует два главных подхода к анализу iOS приложений:

* на джейлбрейкнутом iOS девайсе
* с помощью виртуализации (например, [corellium](https://corellium.com/))

Оба способа вызвали ["негативную реакцию"](https://techcrunch.com/2019/08/15/apple-is-suing-corellium/) в Купертино, где стараются по возможности быстро патчить iOS баги, чтобы сделать невозможным использование джейлбрейк и даже готовы начинать (сомнительные) судебные иски. 

К сожалению, это оставляет исследователей безопасности в некоторой степени привязанными, так как варианты анализа ограничены.

> Apple [пообещала](https://www.theverge.com/2019/8/8/20756629/apple-iphone-security-research-device-program-vulnerabilities) выпустить устройства для разработчиков, которые позволят независимому исследователю безопасности проводить аудит самой iOS и анализировать iOS приложения. Насколько мне известно, эти телефоны еще не попали в руки ученых. Однако, если/когда они это сделают, сообщество будет очень радо этим устройствам. 

Не имея доступа к софту виртуализации, мы "застряли", выполняя анализ ToTok на джейлбрейкнутом телефоне. К счастью, благодаря невероятному инструменту checkra1n мы можем джейлбрейкнуть (и таким образом анализировать iOS приложения) даже последние версии iOS!

![](/assets/images/ru/ios/6.png)

> Подробности о checkm8/checkra1n читайте:
> * ["Developer of Checkm8 explains why iDevice jailbreak exploit is a game changer"](https://arstechnica.com/information-technology/2019/09/developer-of-checkm8-explains-why-idevice-jailbreak-exploit-is-a-game-changer/)
> * ["The One Weird Trick SecureROM Hates"](http://iokit.racing/oneweirdtrick.pdf)
>
> И снова, большое спасибо всем исследователям/хакерам, кто разработал эти замечательные инструменты!

Предположим, что у вас есть доступ к уязвимому устройству (iPhone5x - iPhone X), в таком случае настройка среды анализа достаточно проста и подробно описана в статье [From checkra1n to Frida: iOS App Pentesting Quickstart on iOS 13](https://spaceraccoon.dev/from-checkra1n-to-frida-ios-app-pentesting-quickstart-on-ios-13). Она состоит из следующих шагов:

1. Джейлбрейк с использованием [checkra1n](https://checkra.in/), который устанавливает _Cydia_
2. Настройка _iProxy_
3. Установка остального софта (_Frida_ и т.д.)

Ура! Теперь вы можете начать анализ iOS приложений на вашем джейлбрейкнутом девайсе.

## Анализ ToTok

Теперь давайте перейдем к приложению _ToTok_. 

Так как мы рассматриваем базовый анализ, нашими задачами будет:

* расшифровка приложения
* расшифровка (и мониторинг) сетевого трафика

Для начала, установим _ToTok_ на джейлбрейкнутый девайс:

![](/assets/images/ru/ios/7.png)

Оно установится в папку `/private/var/containers/Bundle/Application` (`<UUID>/ToTok.app`):

![](/assets/images/ru/ios/8.png)

Как было замечено, исполняемый файл главного приложения - зашифрован (с помощью Apple FairPlay DRM). Определить это можно, сдампив загрузочные команды файла `ToTok.app/ToTok` и найдя наличие команды `LC_ENCRYPTION_INFO_64`:

```
otool -Vl ToTok.app/ToTok

...

Load command 12
cmd LC_ENCRYPTION_INFO_64
cmdsize 24
cryptoff 253952
cryptsize 4096
cryptid 1
pad 0
```

> Для более подробной информации о шифровании iOS приложений, читайте: [Copy Protection Overview](https://www.theiphonewiki.com/wiki/Copy_Protection_Overview)

Далее мы кратко обсудим, как полностью расшифровывать приложение, но сейчас у нас осталось еще куча не расшифрованных файлов, на которые надо обратить внимание. 

В первую очередь, мы можем изучить файл `Info.plist`. Он очень большой, приведем лишь часть: 

```xml
$ cat ToTok.app/Info.plist 

<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" ... > 
<plist version="1.0">
<dict>

<key>MinimumOSVersion</key>
<string>13.0</string>

<key>CFBundleIdentifier</key>
<string>ai.totok.videochat</string>

<key>UIBackgroundModes</key>
<array>
<string>audio</string>
<string>fetch</string>
<string>remote-notification</string>
<string>voip</string>
</array>

<key>NSAppTransportSecurity</key>
<dict>
<key>NSAllowsArbitraryLoads</key>
<true/>
</dict>

<key>CFBundleLocalizations</key>
<array>
<string>en</string>
<string>Arabic</string>
<string>zh_CN</string>
<string>zh_TW</string>
</array>

<key>FacebookAppID</key>
<string>1454446651523910</string>

<key>NSMicrophoneUsageDescription</key>
<string>Allow ToTok to access Microphone to make voice messages and 
voice/video calls</string>

<key>NSCalendarsUsageDescription</key>
<string>Allow ToTok to access Calendar to remind you of ToTok Event</string>

<key>NSLocationWhenInUseUsageDescription</key>
<string>Your location is required for providing local weather informatio</string>

<key>NSPhotoLibraryAddUsageDescription</key>
<string>Allow ToTok to access Gallery to send/save pictures</string>

<key>NSContactsUsageDescription</key>
<string>Allow ToTok to read Contacts to find your friends who is using 
ToTok now</string>

<key>NSSiriUsageDescription</key>
<string>You can use Siri to call your ToTok contacts directly.</string>

<key>NSCameraUsageDescription</key>
<string>Allow ToTok to access Camera to take photos and videos</string>
```

Интересные моменты:

* Опция `UIBackgroundModes` сообщает iOS, что приложение должно продолжать работать в фоновом режиме. Подробнее о ` UIBackgroundModes` читайте в [документации](https://developer.apple.com/documentation/bundleresources/information_property_list/uibackgroundmodes?language=objc)
* Ключи `NSAppTransportSecurity/NSAllowsArbitraryLoads` означают, что приложению разрешено общаться по HTTP (обычно iOS разрешает только HTTPS). Подробнее о `NSAllowsArbitraryLoads`, читайте в [документации](https://developer.apple.com/documentation/bundleresources/information_property_list/nsapptransportsecurity/nsallowsarbitraryloads?language=objc) 
* Массив `CFBundleLocalizations` содержит переводы для английского, китайского (`zh_CN`), тайваньского (`zh_CN`) и арабского
* Идентификатор `FacebookAppID (1454446651523910)` по всей видимости, принадлежит компании 'Yeecall' (подробнее об этом далее!)

![](/assets/images/ru/ios/9.png)

* Присутствие различных ключей/значений `*UsageDescription`, сообщает iOS, что отображать, когда приложение запрашивает разрешения. Здесь мы видим, что ToTok запрашивает доступ к: микрофону, календарю, местоположению, фотографиям, контактам, интеграции с Siri и камере.

![](/assets/images/ru/ios/10.png)

Пользователь должен явно разрешить приложению доступ к микрофону, камере и пользовательской информации (фото, местоположение и т.д.)

![](/assets/images/ru/ios/11.png)

...однако такие разрешения нужны для легального функционала приложения, поэтому пользователи всегда соглашаются.

Перед тем, как мы расшифруем приложение, чтобы провести статический анализ, давайте проанализируем сетевой трафик.

## Сетевой трафик

Неудивительно, что весь трафик ToTok зашифрован с помощью SSL. Более того, приложение использует certificate pinning для защиты от MiTM атак. Однако это никак не мешает нам при проведении локального анализа (на джейлбрейкнутом устройстве).

Для начала, мы настраиваем удаленный прокси (например на вашем MacBook). После, на iOS устройстве (в настройках сети), мы указываем адрес нашего прокси, чтобы сообщить iOS, что нужно маршрутизировать через него весь трафик. Лично я использую [Charles](https://www.charlesproxy.com/) прокси ([Burp](https://portswigger.net/burp) тоже подойдет).

> "Charles это HTTP прокси/HTTP монитор/Обратный прокси, который позволяет разработчикам просматривать весь HTTP и SSL/HTTPS трафик между своими компьютерами и интернетом" -https://www.charlesproxy.com/

Существует несколько подробных туториалов, описывающих, как настроить прокси, чтобы читать трафик (даже SSL) вашего iOS устройства. 

* [Charles Proxy Tutorial for iOS](https://www.raywenderlich.com/1827524-charles-proxy-tutorial-for-ios)
* [Using Charles as SSL Proxy on iOS](https://benoitpasquier.com/charles-ssl-proxy-ios/)

Вкратце, нам надо установить прокси Charles и добавить его сертификат в список доверенных:

![](/assets/images/ru/ios/12.png)

Как было сказано, ToTok использует SSL pinning, чтобы защититься от прослушивания трафика. Это значит, что приложение отклонит сертификат прокси и соединение не установится.

Больше информации о SSL pinning, читайте тут: [Preventing Man-in-the-Middle Attacks in iOS with SSL Pinning](https://www.raywenderlich.com/1484288-preventing-man-in-the-middle-attacks-in-ios-with-ssl-pinning)

К счастью, это можно обойти (чтобы читать трафик) установкой и настройкой [SSL Kill Switch 2](https://github.com/nabla-c0d3/ssl-kill-switch2).

![](/assets/images/ru/ios/13.png)

Теперь мы готовы читать зашифрованный трафик!

Запускаем ToTok и видим расшифрованный трафик:

![](/assets/images/ru/ios/14.png)

Большая часть трафика идет на сервер capi.im.totok.ai. Сервер использует самоподписанный сертификат (SHA-256: C9 27 30 CC D5 FE C0 46 3D E8 5A A6 6D FA AB 2F 3B 92 4E 04 C5 1E 0B 6F A4 31 FE 78 33 48 B5 74), выпущенный страной ОАЭ:

![](/assets/images/ru/ios/15.png)

Сетевой трафик довольно типичный для чат-приложения. Посмотрим на некоторые запросы.

Когда приложение запускается первый раз, оно делает GET запрос к capi.im.totok.ai.

Например:

```http
https://capi.im.totok.ai/uc/state/AABqbEEgqCA=/white?
n=cJECAUMZzcI=&c=iJXfhRcO4JU=&did=3yDQkh1qf8iai1UKOGNUnr8sJIo=&model=iPhone
&pkg=ai.totok.videochat&clientver=1.2.9&loc=en-US_US
```

Разобьем по параметрам:

* n: cJECAUMZzcI=
* c: iJXfhRcO4JU=
* did: 3yDQkh1qf8iai1UKOGNUnr8sJIo=
* model: iPhone
* pkg: ai.totok.videochat
* clientver: 1.2.9
* loc: en-US_US

Сервер отвечает:

```json
{
"response": {
"white": false
},
"responseHeader": {
"status": 200,
"version": "1.0"
}
}
```

Другой GET запрос, тоже к capi.im.totok.ai, идет на /api/idc/find, который возвращает:

```json
{
"response": {
"uc": "capi.im.totok.ai",
"pfm": "capi.im.totok.ai",
"doodle": "in.debug.yeecall.com:8080",
"uc_ssl": "capi.im.totok.ai",
"pfm_ssl": "capi.im.totok.ai",
"doodle_ssl": "capi.im.totok.ai",
"idcIdx": 1,
"wallet_ssl": "capi.im.totok.ai"
},
"responseHeader": {
"status": 200,
"version": "1.0"
}
```

... еще одна отсылка к yeecall.

Продолжаем: после верификации пользователя, приложение скачивает фото пользовательского профиля по адресу https://ucdefault-sg.oss-ap-southeast-1.aliyuncs.com/

![](/assets/images/ru/ios/16.png)

Сайт aliyuncs.com является частью Alibaba Cloud.

Если пользователь разрешил приложению ToTok доступ к контактам, то приложение пытается отправить всю адресную книгу POST запросом:

https://capi.im.totok.ai/log/report/report?n=cJECAUy9BHI=&c=iJXfhRiqKSU=&did=3yDQkh1qf8iai1UKOGNUnr8sJIo=&model=...

![](/assets/images/ru/ios/17.png)

```json
{
"appver": "1.2.9",
"os": "iOS",
"time": 1576888571519.1421,
"app": "ToTok",
"event": "contact",
"uid": "AAAAAs7FxJY=",
"data": {
"version": "1.0",
"contacts": [{
"familyName": "Smith",
"birthday": 0,
"modifyDate": 1576134814000,
"nickname": "",
"displayName": "John Smith",
"organizationName": "Apple",
"departmentName": "",
"namePrefix": "",
"nameSuffix": "",
"id": 1,
"primaryIDs": [{
"value": "(808) 265-3214",
"label": "_$!<Home>!$_"
}],
"middleName": "",
"jobTitle": "",
"contactType": 0,
"note": "",
"phoneticMiddleName": "",
"phoneticGivenName": "",
"emailAddresses": [],
"createDate": 1576134814000,
"phoneticFamilyName": "",
"givenName": "John"
}, {
"familyName": "Turner",
"birthday": 0,
"modifyDate": 1576136649000,
"nickname": "",
"displayName": "Tina Turner",
"organizationName": "Apple",
"departmentName": "",
"namePrefix": "",
"nameSuffix": "",
"id": 2,
"primaryIDs": [{
"value": "(814) 523-4155",
"label": "_$!<Home>!$_"
}],
"middleName": "",
"jobTitle": "",
"contactType": 0,
"note": "",
"phoneticMiddleName": "",
"phoneticGivenName": "",
"emailAddresses": [],
"createDate": 1576136649000,
"phoneticFamilyName": "",
"givenName": "Tina"
}]
},
"language": "en",
"network": "WiFi",
"osver": "13.1.3"
}
```

![](/assets/images/ru/ios/18.png)

Вы, конечно, можете сказать, что это безобидный функционал приложения, необходимый для связи со своими друзьями 

Когда пользователи обмениваются медиа контентом (например, фото), отсылается POST запрос (с содержимым файла) для загрузки на https://capi.im.totok.ai/pfm/up/AABqbEEgqCA=/sendtouser. Сервер возвращает информацию об успешном получении файла:

```json
{
"response": {
"download": {
"url": "https://capi.im.totok.ai/pfm/download/file?fid=5dfe1lqtkl9555884b0b0001ae7119 ...
"size": 81664,
"fid": "5dfe1lqtkl9555884b0b0001ae7119",
"md5": "a3b89da4ccb998ef9fa66fb11bafcfa1"
}
},
"responseHeader": {
"status": 200,
"version": "1.0"
}
}
```

Перед закачкой, файл шифруется и упаковывается:

```hex
00000000 73 d6 6d 3c be 5a d0 43 b6 cb 23 b1 46 b2 c2 b7 |s.m<.Z.C..#.F...|
00000010 16 71 9b f4 67 51 63 0f 7b 8e d7 44 6c 85 77 f7 |.q..gQc.{..Dl.w.|
00000020 ef 9d 0d b3 a3 2e 27 2d e6 ca 84 7f 9d cc ec 30 |......'-.......0|
00000030 44 00 aa 00 ca 90 f1 c2 d7 48 c7 c9 f8 8c 30 9d |D........H....0.|
00000040 8b 4e b0 8c 03 16 e6 3f 25 20 25 e2 44 29 19 1d |.N.....?% %.D)..|
00000050 49 98 9d 9c 0e 23 ae 8a 54 30 1c fd 7e 2a 6e 76 |I....#..T0..~*nv|
00000060 cf 2b a7 cd 26 88 06 8d 42 15 57 a2 96 4a e3 2c |.+..&...B.W..J.,|
00000070 fc 9f 7f e7 92 cf 06 d0 ef 8e fe fe a6 a4 e4 7f |................|
00000080 d3 88 20 2d e5 77 56 4d 32 7b c7 b0 6b 17 04 e0 |.. -.wVM2{..k...|
00000090 7c e4 18 9f b2 e9 56 90 2c af dd c8 9b 35 f5 e2 ||.....V.,....5..|
000000a0 f7 03 f8 1b 2f 6d a7 ef 91 88 f8 80 82 74 2e ec |..../m.......t..|
000000b0 44 61 b3 d2 5d 3a a3 92 ef c4 48 d9 08 bb 29 e6 |Da..]:....H...).| 
```

На данный момент неизвестно, поддерживает ли приложение end-to-end шифрование, и если да, то является ли это достаточным.

Политика конфиденциальности ToTok гласит:

"Сообщения: все данные хранятся в зашифрованном виде, чтобы инженеры ToTok или злоумышленники не могли получить доступ"

... но похоже это относится только "хранимым данным"?

Наконец, если пользователь разрешил доступ к местоположению, приложение делает запрос к meiduoyun.ws.amberweather.com:

https://meiduoyun.ws.amberweather.com/api/v1/weather?appid=12027&lang=en&lat=25.276987&lon=55.296249&pkg=ai.totok.videochat...

Заметьте, это запрос содержит точную геолокацию пользователя (lang=en&lat=25.276987&lon=55.296249).

## Расшифровка приложения

Чтобы провести статический анализ ToTok'а, нам надо расшифровать его исполняемый файл.

Для этого мы будем использовать [Clutch](https://github.com/KJCracks/Clutch) - высокоскоростной iOS расшифровщик. Эта утилита берет bundle id iOS приложения (например ai.totok.videochat) и генерирует полностью расшифрованный файл .ipa!

```
root# Clutch -d ai.totok.videochat
Clutch[10196:1254709] command: Dump specified bundleID into .ipa file

Zipping ToTok.app
Dumping (arm64)
Patched cryptid (64bit segment)
Dumping (arm64)

...
Successfully dumped framework ToTokCommonBase!
Dumping arm64
Successfully dumped framework PINRemoteImage!
Dumping arm64
Dumping arm64

...
Zipping ToTokCommonBase.framework
Zipping External.framework
Zipping ToTok3rdSDK.framework
Zipping ShareExtension.appex
DONE: /private/var/mobile/Documents/Dumped/ai.totok.videochat-iOS13.0-(Clutch-(null)).ipa
Finished dumping ai.totok.videochat in 9.3 seconds
```

Другой способ получить исполняемый файл приложения, это использовать [Fridump](https://github.com/Nightbringer21/fridump). Если запустить с опцией `-s`, то мы получим все строки в файлах:

```
patrick$ python fridump.py -U -s ToTok

______ _ _
| ___| (_) | |
| |_ _ __ _ __| |_ _ _ __ ___ _ __
| _| '__| |/ _` | | | | '_ ` _ \| '_ \
| | | | | | (_| | |_| | | | | | | |_) |
\_| |_| |_|\__,_|\__,_|_| |_| |_| .__/
| |
|_|
Current Directory: /Users/patrick/Downloads/fridump-master
Output directory is set to: /Users/patrick/Downloads/fridump-master/dump
Creating directory...
Starting Memory dump...

Running strings on all files:
Progress: [##################################################] 100.0% Complete

Finished!
```

Туториал по использованию Fridump: [Fridump – iOS Examples](http://pentestcorner.com/fridump-ios-examples/)

В любом случае теперь мы имеем расшифрованное приложение и можем изучать его код. Большой размер приложения и фреймворков (25 Мб+) накладывают временные ограничения на анализ.

При анализе бинарника, один из первых шагов это экспортировать содержащиеся в нем строки и классы, с помощью [Class-dump](http://stevenygard.com/projects/class-dump/). Обычно это дает много информации о функционале приложения и/или других интересных вещах.

Изучая строки и классы, мы получаем некоторое представление о возможной истории происхождения приложения:

/Users/jiangyaguang/Downloads/totok/YeeCall/Classes/TKSystemInfo.m /Users/jiangyaguang/Downloads/totok/YeeCall/Classes/Call/TKCallSessionViewController.m

```
patrick$ class-dump ToTok.app/ToTok

@interface YeecallContact : NSManagedObject
{
}

@interface YeecallReminder : NSManagedObject
{
}

@interface YeeCallSecurityPolicy : AFSecurityPolicy
{
}
```

Как уже отмечалось ранее, мы обнаружили другие связи с YeeCall. Из исследования строк в приложении ясно, что ToTok в основном состоит из кода [YeeCall](http://www.yeecall.com/en/index.html). По словам [CrunchBase](https://www.crunchbase.com/organization/yeecall), YeeCall - это "компания-разработчик программного обеспечения, которая разработала мессенджер Yeecall для видео- и голосовых звонков". Неудивительно, что ToTok просто основан на существующем коде/продукте (а не написан полностью с нуля).

Возможно, что "Breej Holding Ltd" ("издатель" приложения для iOS) просто заключил контракт или лицензировал существующий код с "YeeCall" для создания приложения ToTok. Это был бы простой и эффективный способ быстро создать новое полнофункциональное приложение. Это также объясняет довольно странные CFBundleLocalizations - <string>Arabic</string>, <string>zh_CN</string> и <string>zh_TW</string>.

Среди строк бинарника, есть строка static.totok.ai. Это сервер, который обслуживает статический контент для приложения (изображения, иконки и т.д.). Сервер, кажется, неправильно настроен, и, как следствие, просмотр его корня показывает список всех размещенных файлов:

![](/assets/images/ru/ios/19.png)

## Кто это сделал?

Перед тем, как закончить, давайте кратко обсудим, кто может стоять за этим приложением, так как в каком-то смысле это самый насущный вопрос.

Во-первых, достаточно ясно, что "Breej Holding Ltd" не является фактическим iOS разработчиком или издательской компанией:

> _"Технический анализ и интервью с экспертами по компьютерной безопасности показали, что фирма, стоящая за ToTok, Breej Holding, скорее всего, является подставной компанией, аффилированной с DarkMatter, базирующейся в Абу-Даби киберразведкой и хакерской фирмой, в которой работают сотрудники разведки Эмирата" - New York Times_

![](/assets/images/ru/ios/20.png)

Хотя газета Times не вдается в (гораздо) более детальное описание этого заявления, недавно [Bill Marczak](https://twitter.com/billmarczak) (научный сотрудник лаборатории [CitizenLab](https://citizenlab.ca/)) опубликовал невероятно подробный отчет о "корпоративной структуре, стоящей за ToTok":

> НОВОСТЬ: Я глубоко погрузился в корпоративную структуру, стоящую за приложением ToTok VoIP. Классифицированная оценка разведки США (по данным NYT) говорит о том, что ToTok - это шпионский инструмент, разработанный разведкой ОАЭ. https://t.co/HLMcwWMCZn.
>
> - Bill Marczak (@billmarczak) [2 января, 2020](https://twitter.com/billmarczak/status/1212800557556985856?ref_src=twsrc%5Etfw)

Читать подробнее: ["A BREEJ TOO FAR: How Abu Dhabi’s Spy Sheikh hid his Chat App in Plain Sight"](https://medium.com/@billmarczak/how-tahnoon-bin-zayed-hid-totok-in-plain-sight-group-42-breej-4e6c06c93ba6)

В своей [статье](https://medium.com/@billmarczak/how-tahnoon-bin-zayed-hid-totok-in-plain-sight-group-42-breej-4e6c06c93ba6), он довольно подробно раскрывает многочисленных участников и компании, стоящие за ToTok, включая тех, кто "...связан с шейхом Tahnoon bin Zayed al-Nahyan, "старшим должностным лицом разведки ОАЭ"". 

Следующая диаграмма четко визуализирует эти связи (спасибо за изображение: [Bill Marczak](https://twitter.com/billmarczak)):

![](/assets/images/ru/ios/21.png)

> _"ToTok, похоже, является новым случаем цифровой платформы, тайно управляемой национальным государством, чтобы получить стратегическое преимущество в сборе разведданных". -Bill Marczak_

## Заключение

В этом посте мы исследовали ToTok -- приложение для iOS, которое, как утверждают американские разведчики, было шпионским инструментом, используемым правительством Объединенных Арабских Эмиратов. 

Наш анализ показал, что ToTok просто делает то, что и заявляет... и на самом деле ничего больше. Предположив, что ToTok действительно предназначен для шпионажа за своими пользователями, эта "законная" функциональность приложения, на самом деле, является гением всей операции массового наблюдения: никаких эксплойтов, никаких бэкдоров, никаких вредоносных программ, ...опять же, просто "законная" функциональность, которая, по всей видимости, позволила сформировать четкое представление о большом проценте населения страны.

Подумайте об этом так:
...вы - (скорее слежка) иностранное правительство, которое с удовольствием следило бы за вашими гражданами.

Пять простых шагов:

1. Запретите популярные приложения, такие как WhatsApp, Skype...
2. Создайте бесплатное альтернативное приложение, предоставляющее этот запретный функционал.
3. Отправьте приложение в iOS app store, где оно будет легко одобрено Apple.
4. Создавайте фальшивые отзывы и сообщения в социальных сетях, рекомендующие приложение.
5. Подождите, пока граждане вашей страны с готовностью примут приложение, и его популярность взлетит до небес.

...ура! Теперь у вас есть доступ к адресным книгам пользователей, чатам, местоположению и многому другому, в совершенно "законном", утвержденном Apple, виде!

Такая коллекция обеспечивает достаточную "фазу 1" (так же, как и программа АНБ по сбору больших метаданных) более комплексной операции по сбору разведданных. Как только вы узнаете, кто с кем разговаривает, и, возможно, даже то, что они говорят, вы сможете определить конкретных людей, представляющих интерес, и нацелить на них более продвинутые инструменты. Эта "фаза 2" включает в себя более традиционные наступательные кибер-операции: более целенаправленные, скрытные и инвазивные. Однако такая фаза гораздо дороже и труднее поддается масштабированию, а потому требует наличия достаточного "компонента фазы 1" ... например, ToTok.

Для получения более подробной информации о "фазе 2" кибер-операций на Ближнем Востоке, ознакомьтесь с: ["Inside the UAE’s Secret Hacking Team Of American Mercenaries"](https://www.reuters.com/investigates/special-report/usa-spying-raven/)
