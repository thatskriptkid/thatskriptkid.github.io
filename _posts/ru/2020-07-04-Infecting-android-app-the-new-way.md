---
layout: post
title: Новый способ внедрения вредоносного кода в андроид приложения
tags: [android, malware]
category: [ru]
---

![](/assets/images/infecting_android/silent_night.png)

- [Предисловие](#предисловие)
- [Недостатки текущего подхода](#недостатки-текущего-подхода)
- [Описание нового подхода](#описание-нового-подхода)
- [Преимущества нового подхода](#преимущества-нового-подхода)
- [Выявление необходимых модификаций в AndroidManifest.xml и патчинг](#выявление-необходимых-модификаций-в-androidmanifestxml-и-патчинг)
- [Создание файлов, для внедрения в целевое приложение](#создание-файлов-для-внедрения-в-целевое-приложение)
- [Выявление необходимых модификаций в DEX и патчинг](#выявление-необходимых-модификаций-в-dex-и-патчинг)
- [Результаты](#результаты)
- [Ограничения нового подхода](#ограничения-нового-подхода)
- [Дальнейшие улучшения PoC](#дальнейшие-улучшения-poc)
- [FAQ](#faq)

## Предисловие

**Авторы идеи: Ербол & Thatskriptkid**

**Автор рисунка: @alphin.fault [instagram](https://www.instagram.com/alphin.fault)**

**Автор статьи и proof-of-concept кода: Thatskriptkid**

**[Proof-of-Concept](https://github.com/thatskriptkid/apk-infector-Archinome-PoC)**

Целевая аудитория статьи - люди, которые имеют представление о текущем способе заражения андроид приложений через патчинг 
smali кода и хотят узнать о новом и более эффективном способе. Если вы не знакомы с текущей практикой заражения, то прочитайте мою статью - [Воруем эцп, используя Man-In-The-Disk]({% post_url ru/2019-07-01-man-in-the-disk %}), глава - "Создаем payload". Техника описанная здесь, полностью была придумана нами, в сети отсутствует какое-либо описание подобного способа. 

Наш способ:
1. Не использует баги или уязвимости андроида
2. Не предназначен для крякинга приложений (удаление рекламы, лицензии и т.д.) 
3. Предназначен для добавления вредоносного кода, без какого-либо вмешательства в работу целевого приложения или его внешний вид.

## Недостатки текущего подхода

Способ внедрения вредоносного кода, с помощью декодирования приложения до smali кода и его патчинг - является единственным и широко практикуемым на сегодняшний день. [smali/backsmali](https://github.com/JesusFreke/smali) - единственный инструмент, используемый для этого. На основе него строятся все известные инфекторы, например:

1. [backdoor-apk](https://github.com/dana-at-cp/backdoor-apk)
2. [TheFatRat](https://github.com/Screetsec/TheFatRat)
3. [apkwash](https://github.com/jbreed/apkwash)
4. [kwetza](https://github.com/sensepost/kwetza)

Малварь также использует smali/backsmali и патчинг. Схема работы трояна [Android.InfectionAds.1](https://vms.drweb.com/virus/?i=17771929&lng=en):

![](/assets/images/infecting_android/img_for_paper/android_infection_ads_01_en.png)

Декодирование и патчинг предполагают изменение оригинального classesN.dex файла. Это приводит к двум проблемам:

1. Выход за пределы [лимита в 65536](https://developer.android.com/studio/build/multidex#about) методов [в одном DEX файле](https://github.com/JesusFreke/smali/issues/629), если вредоносного кода слишком много
2. Приложение может проверять целостность DEX файлов

Декодирование/дизассемблирование DEX - это сложный процесс, требующий постоянного обновления и [сильно зависящий от версии андроида](https://github.com/JesusFreke/smali/issues/595). 

Практически все доступные инструменты для заражения/модификации написаны на Java и/или зависят от JVM - это сильно сужает область использования и делает невозможным запуск инфектора на роутерах, встроенных системах, системах без JVM и т.д.

## Описание нового подхода

В андроиде существует несколько типов запуска приложений, один из них называется cold start. Cold start - запуск приложения впервые. 

![](/assets/images/infecting_android/img_for_paper/cold_launch.png)

Выполнение приложения начинается с создания Application объекта. Большинство андроид приложений имеют свой Application класс, который должен наследоваться от основного класса `android.app.Applciation`. Пример класса:

```java
package test.pkg;
import android.app.Application;
public class TestApp extends Application {

    public TestApp() {}

    @Override
    public void onCreate() {
        super.onCreate();
    }
}
```

Класс `test.pkg.TestApp` должен быть прописан в AndroidManifest.xml. Пример манифеста:

```xml
<?xml version="1.0" encoding="utf-8"?>
<manifest xmlns:android="http://schemas.android.com/apk/res/android"
    package="com.example">

    <application
        android:icon="@mipmap/ic_launcher"
        android:label="Test"
        android:roundIcon="@mipmap/ic_launcher_round"
        android:name="test.pkg.TestApp">
    </application>
</manifest>
```

Процесс запуска такого приложения:

![](/assets/images/infecting_android/img_for_paper/Usual_app_launch.png)

Были определены основные требования к нашей технике заражения:

1. Выполнение вредоносного кода, при старте приложения 
2. Сохранение всех этапов процесса запуска оригинального приложения
   
Внедрение вредоносного кода происходило в стадии алгоритма *cold start*:*Application Object creation->Application Object Constructor*. Был создан вредоносный Application класс, внедрен в приложение и прописан в *AndroidManifest.xml*, вместо изначального. Чтобы сохранять прежнюю цепочку выполнения, он был наследован от `test.pkg.TestApp`.

Вредоносный Application класс:

```java
package my.malicious;
import test.pkg;
public class InjectedApp extends TestApp {

    public InjectedApp() {
        super();
        executeMaliciousPayload();
    }
}
```

Модифицированный AndroidManifest.xml:

```xml
<?xml version="1.0" encoding="utf-8"?>
<manifest xmlns:android="http://schemas.android.com/apk/res/android"
    package="com.example">

    <application
        android:icon="@mipmap/ic_launcher"
        android:label="Test"
        android:roundIcon="@mipmap/ic_launcher_round"
        android:name="my.malicious.InjectedApp">
    </application>
</manifest>
```

Процесс запуска вредоносного кода внутри зараженного приложения (красным выделены модификации):

![](/assets/images/infecting_android/img_for_paper/Infected_app_launch.png)

Примененные модификации:

1. В приложение добавлен класс `my.malicious.InjectedApp`
2. В AndroidManifest.xml заменена строка `test.pkg.TestApp` на `my.malicious.InjectedApp`

## Преимущества нового подхода

Существует возможность применить необходимые модификации к APK:

1. Без дизассемблирования/сборки DEX
2. Без декодирования/кодирования манифеста
3. Без внесения изменений в оригинальные DEX файлы

Данные факты позволяют заражать практически любое существующее приложение, без ограничений. Добавление своего класса и изменение манифеста работает намного быстрее, чем декодирование. Внедренный нашей техникой вредоносный код стартует сразу, а не по определенному событию, так как мы внедряемся в самое начало процесса запуска приложения. Описанная техника заражения не зависит от архитектуры и версии андроида (за небольшим исключением). 

PoC для демонстрации был написан на Go и готов к расширению до полноценного инструмента. PoC компилируется в один целостный бинарный файл и не использует никаких зависимостей в рантайме. Использование Go позволяет, с помощью кросс-компиляции, собрать инфектор для практически любой архитектуры и ОС.

Тестирование приложений, зараженных PoC проводилось на:

```
NOX player 6.6.0.8006-7.1.2700200616, Android 7.1.2 (API 25), ARMv7-32

NOX player 6.6.0.8006-7.1.2700200616, Android 5.1.1 (API 22), ARMv7-32

Android Studio Emulator, Android 5.0 (API 21), x86

Android Studio Emulator, Android 7.0 (API 24), x86

Android Studio Emulator, Android 9.0 (API 28), x86_64

Android Studio Emulator, Android 10.0 (API 29), x86

Android Studio Emulator, Android 10.0 (API 29), x86_64

Android Studio Emulator, Android API 30, x86

Xiaomi Mi A1
```

Удалось удачно заразить огромное количество приложений (по понятным причинам имена скрыты). Удалось заразить приложения, которые не поддаются декодированию, с помощью smali/backsmali, а значит и любого существующего инструмента.

## Выявление необходимых модификаций в AndroidManifest.xml и патчинг

Одной из модификаций, необходимой для заражения, является замена строки в AndroidManifest.xml. Существует возможность пропатчить строку, без декодирования/кодирования манифеста. 

APK содержат манифест в бинарном, закодированном виде. Структура бинарного манифеста не документирована и представляет собой кастомный алгоритм кодирования XML от Google. Для удобства [было создано](https://github.com/thatskriptkid/Kaitai-Struct-Android-Manifest-binary-XML) описание на языке [Kaitai Struct](https://kaitai.io/), которое может быть использовано, а качестве документации.

Структура AndroidManifest.xml (в скобках - размер в байтах):

![](/assets/images/infecting_android/img_for_paper/Manifest_draw.png)

Для определения изменений в манифесте, после патчинга оригинального имени Application класса на вредоносное, были разработаны два приложения, с разными имена классов. Приложения были собраны в APK и распакованы, для получения бинарных манифестов.

Пример оригинального манифеста, с именем Application - `test.pkg.TestApp`:

```xml
<?xml version="1.0" encoding="utf-8"?>
<manifest xmlns:android="http://schemas.android.com/apk/res/android"
    package="com.qoogle.service.outbound.thread.safe.eng.packages.packas.pack.level.random">

    <application
        android:icon="@mipmap/ic_launcher"
        android:label="MinDEX"
        android:roundIcon="@mipmap/ic_launcher_round"
        android:name="test.pkg.TestApp">
    </application>

</manifest>
```

Пример пропатченного манифеста, с именем Application - `test.pkg.TestAppAAAAAAAAA`:

```xml
<?xml version="1.0" encoding="utf-8"?>
<manifest xmlns:android="http://schemas.android.com/apk/res/android"
    package="com.qoogle.service.outbound.thread.safe.eng.packages.packas.pack.level.random">

    <application
        android:icon="@mipmap/ic_launcher"
        android:label="MinDEX"
        android:roundIcon="@mipmap/ic_launcher_round"
        android:name="test.pkg.TestAppAAAAAAAAA">
    </application>

</manifest>
```

Длина полного имени класса увеличилась на 9 символов. Оба файла были открыты в [HexCmp](https://www.fairdell.com/hexcmp/), для получения диффа. 

Изменения, которым подвергся манифест и объяснение причин:

<style type="text/css">
.tg  {border-collapse:collapse;border-spacing:0;}
.tg td{border-color:black;border-style:solid;border-width:1px;font-family:Arial, sans-serif;font-size:14px;
  overflow:hidden;padding:10px 5px;word-break:normal;}
.tg th{border-color:black;border-style:solid;border-width:1px;font-family:Arial, sans-serif;font-size:14px;
  font-weight:normal;overflow:hidden;padding:10px 5px;word-break:normal;}
.tg .tg-0pky{border-color:inherit;text-align:left;vertical-align:top}
</style>
<table class="tg">
<thead>
  <tr>
    <th class="tg-0pky">field</th>
    <th class="tg-0pky">offset</th>
    <th class="tg-0pky">description</th>
    <th class="tg-0pky">diff_count</th>
    <th class="tg-0pky">explanation</th>
  </tr>
</thead>
<tbody>
  <tr>
    <td class="tg-0pky">header.file_len</td>
    <td class="tg-0pky">0x4</td>
    <td class="tg-0pky">Длина всего файла</td>
    <td class="tg-0pky">0x10</td>
    <td class="tg-0pky">В оригинальном манифесте было 0х2 байта выравнивания, в измененном они не требуются. <br>Строки в бинарном манифесте хранятся в формате UTF-16, то есть один символ занимает 0x2 байта.<br>Итого, мы увеличили строку на 9 символов (0x12 байт) минус 0x2 байта выравнивания, получаем разницу 0x10 байт</td>
  </tr>
  <tr>
    <td class="tg-0pky">header.string_table_len</td>
    <td class="tg-0pky">0xC</td>
    <td class="tg-0pky">Длина массива строк</td>
    <td class="tg-0pky">0x10</td>
    <td class="tg-0pky">Строка находится в общем массиве строк. Объяснение разницы в 0x10 байт такая же как у header.file_len</td>
  </tr>
  <tr>
    <td class="tg-0pky">string_offset_table.offset</td>
    <td class="tg-0pky">0x7C</td>
    <td class="tg-0pky">Оффсет до строки, следующей после измененной</td>
    <td class="tg-0pky">0x12</td>
    <td class="tg-0pky">В string_offset_table хранятся оффсеты до строк в массиве строк манифеста. Так как длина строки увеличилась, <br>следующая за ней строка сдвинулась дальше на 0x12 байт. Выравниваниеи здесь не учитывается, так как оффсеты<br>расположены до массива строк.</td>
  </tr>
</tbody>
</table>

![](/assets/images/infecting_android/img_for_paper/Manifest_diff_1.png)

<style type="text/css">
.tg  {border-collapse:collapse;border-spacing:0;}
.tg td{border-color:black;border-style:solid;border-width:1px;font-family:Arial, sans-serif;font-size:14px;
  overflow:hidden;padding:10px 5px;word-break:normal;}
.tg th{border-color:black;border-style:solid;border-width:1px;font-family:Arial, sans-serif;font-size:14px;
  font-weight:normal;overflow:hidden;padding:10px 5px;word-break:normal;}
.tg .tg-0pky{border-color:inherit;text-align:left;vertical-align:top}
</style>
<table class="tg">
<thead>
  <tr>
    <th class="tg-0pky">field</th>
    <th class="tg-0pky">offset</th>
    <th class="tg-0pky">description</th>
    <th class="tg-0pky">diff_count</th>
    <th class="tg-0pky">explanation</th>
  </tr>
</thead>
<tbody>
  <tr>
    <td class="tg-0pky">strings.len</td>
    <td class="tg-0pky">0x2EA</td>
    <td class="tg-0pky">Длина строки</td>
    <td class="tg-0pky">0x9</td>
    <td class="tg-0pky">Количество символов, на которое увеличилась строка</td>
  </tr>
</tbody>
</table>

![](/assets/images/infecting_android/img_for_paper/Manifest_diff_2.png)

В структуре манифеста, приведенной в начале, после `strings` следует `padding`, для выравнивания `resource_header`. В оригинальном манифесте последняя строка `uses-sdk` заканчивается
по оффсету `0x322` (оранжевым), а значит были добавлены два байта выравнивания (зеленым) для `resource_header`

![](/assets/images/infecting_android/img_for_paper/Manifest_alignment_original.png)

В модифицированном варианте, `string_table` заканчивается на оффсете `0x334` (оранжевым) и далее сразу следует `resource_header` (красным), который не требует выравнивания.

![](/assets/images/infecting_android/img_for_paper/Manifest_alignment_longer.png)

Структура AndroidManifest.xml, с указанием полей, которые необходимо пропатчить, для замены имени оригинального Applciation класса на вредоносный (выделены красным):

![](/assets/images/infecting_android/img_for_paper/Manifest_draw_modified.png)

Proof-of-Concept код, разработанный для статьи, реализовывает эти модификации в методе `manifest.Patch()`.

## Создание файлов, для внедрения в целевое приложение

Второй модификацией, необходимой для заражения, является внедрение класса, с вредоносным кодом. Для сохранения оригинальной цепочки запуска приложения, в него должен быть внедрен Application класс, родительским классом которого должен является оригинальный Applciation класс. На этапе подготовке внедряемых файлов, оно неизвестно. Поэтому, при создании класса, необходимо было использовать имя-заглушку `z.z.z`. 

Изначальное состояние приложения и внедряемого DEX:

![](/assets/images/infecting_android/img_for_paper/DEX_initial_state.png)

После получения оригинального имени Application класса из манифеста, заглушка была пропатчена:

![](/assets/images/infecting_android/img_for_paper/DEX_state_after_patching.png)

Процедура заражения завершается добавлением вредоносного DEX в целевое приложение:

![](/assets/images/infecting_android/img_for_paper/DEX_state_after_injection.png)

Так как классы с вредоносным кодом могут иметь разный код, они были вынесены в отдельный DEX. Это также было сделано для упрощения процедуры патчинга заглушки.

![](/assets/images/infecting_android/img_for_paper/DEX_split.png)

Имена классов в DEX располагаются в алфавитном порядке. Имя Application класса целевого приложения может начинаться с любой буквы. Для предсказуемости порядка строк, после патчинга, имя заглушки было выбрано равным `z.z.z`. 

![](/assets/images/infecting_android/img_for_paper/DEX_strings.png)

Для подготовки внедряемых файлов, был создан проект в Android Studio, с тремя классами.

Класс `InjectedApp`. Его полное имя:

`aaaaaaaa.aaaaaaaaaaaaaaaaaaaa.aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa.aaaaaaaaaaaaaaaaaaaaaa.InjectedApp`

Это имя должно удовлетворять двум правилам:

1. Оно должно быть длиннее любого имени Application класса любого целевого приложения

2. Оно должно быть выше в алфавитном порядке любого имени Application класса любого приложения

Класс `InjectedApp`, который будет выполняться вместо Application class целевого приложения:

```java
package aaaaaaaa.aaaaaaaaaaaaaaaaaaaa.aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa.aaaaaaaaaaaaaaaaaaaaaa;
import aaaaaaaaaaaa.payload;
import z.z.z;

public class InjectedApp extends z {

    public InjectedApp() {
        super();
        payload p = new payload();
        p.executePayload();
    }
}
```

Задача класса начать выполнение вредоносного кода, который находится в другом DEX:

```java
        payload p = new payload();
        p.executePayload();
```

Класс `payload` содержит вредоносный код:

```java
package aaaaaaaaaaaa;

import android.util.Log;

public class payload {

    public void executePayload() {
            Log.i("HELL", "Hello, I'm a malicious payload");
    }
}
```

Полное имя класса должно удовлетворять следующему правилу:

1. Оно должно быть выше по алфавиту любого имени класса Application любого приложения 

Для внедрения произвольного вредоносного кода необходимо создать DEX файл, который должен соблюдать условия:

1. Содержать класс с именем:

`aaaaaaaaaaaa.payload`

2. Класс должен содержать метод

`public void executePayload()`

Класс-заглушка `z.z.z`, полное имя которого будет пропатчено на полное имя Applciation класса целевого приложения.

```java
package z.z;

import android.app.Application;

public class z extends Application {
}
```

Класс должен соблюдать условие:

1. Полное имя класса должно быть ниже по алфавиту полных имен классов `InjectedApp` и `payload`. 
2. Полное имя класса должно быть короче любых полных имен Application классов любых приложений. 

В соответствии с разработанной схемой внедрения, классы `InjectedApp` и `payload` были скомпилированы в отдельные DEX. Для этого в Android Studio была выполнена сборка APK командой `Android Studio->Generate Signed Bundle/APK->release`. Скомпилированные .class файлы создались в папке `app\build\intermediates\javac\release\classes`.

Компилирование .class файлов в DEX, с помощью [d8](https://developer.android.com/studio/command-line/d8):

`d8 --release --min-api 16 --no-desugaring InjectedApp.class --output .`

`d8 --release --min-api 16 --no-desugaring payload.class --output .`

Получившиеся DEX должны быть добавлены в целевое приложение.

## Выявление необходимых модификаций в DEX и патчинг

После патчинга заглушки `z.z.z` на полное имя Application класса целевого приложения, структура DEX изменится. Для выявления модификаций, в Android Studio было создано два приложения с именами классов разной длины.

Класс `InjectedApp`, наследуемый от `z.z.z`, в первом приложении:

```java
package aaaaaaaa.aaaaaaaaaaaaaaaaaaaa.aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa.aaaaaaaaaaaaaaaaaaaaaa;
import aaaaaaaaaaaa.payload;
import z.z.z;

public class InjectedApp extends z {

    public InjectedApp() {
        super();
        payload p = new payload();
        p.executePayload();
    }
}
```

Класс `InjectedApp`, наследуемый от `z.z.zzzzzzzzzzzzzzzz` во втором приложении:

```java
package aaaaaaaa.aaaaaaaaaaaaaaaaaaaa.aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa.aaaaaaaaaaaaaaaaaaaaaa;
import aaaaaaaaaaaa.payload;
import z.z.z;

public class InjectedApp extends zzzzzzzzzzzzzzzz {

    public InjectedApp() {
        super();
        payload p = new payload();
        p.executePayload();
    }
}
```

Длина имени класса увеличилась на 15 символов. Компилируем оба класса отдельно в DEX:

`d8 --release --min-api 16 --no-desugaring InjectedApp.class --output .`

Откроем получившиеся DEX в программе HexCMP:

*[Официальная документация по структуре DEX](https://source.android.com/devices/tech/dalvik/dex-format)*

<style type="text/css">
.tg  {border-collapse:collapse;border-spacing:0;}
.tg td{border-color:black;border-style:solid;border-width:1px;font-family:Arial, sans-serif;font-size:14px;
  overflow:hidden;padding:10px 5px;word-break:normal;}
.tg th{border-color:black;border-style:solid;border-width:1px;font-family:Arial, sans-serif;font-size:14px;
  font-weight:normal;overflow:hidden;padding:10px 5px;word-break:normal;}
.tg .tg-0pky{border-color:inherit;text-align:left;vertical-align:top}
.tg .tg-0pky{text-align:left;vertical-align:top}
</style>
<table class="tg">
<thead>
  <tr>
    <th class="tg-0pky">field</th>
    <th class="tg-0pky">offset</th>
    <th class="tg-0pky">description</th>
    <th class="tg-0pky">diff_count</th>
    <th class="tg-0pky">explanation</th>
  </tr>
</thead>
<tbody>
  <tr>
    <td class="tg-0pky">header_item.checksum</td>
    <td class="tg-0pky">0x8</td>
    <td class="tg-0pky">Контрольная сумма</td>
    <td class="tg-0pky">full</td>
    <td class="tg-0pky">При любом изменении DEX контрольная сумма пересчитывается</td>
  </tr>
  <tr>
    <td class="tg-0pky">header_item.signature</td>
    <td class="tg-0pky">0xC</td>
    <td class="tg-0pky">Хэш</td>
    <td class="tg-0pky">full</td>
    <td class="tg-0pky">При любом изменении DEX хэш пересчитывается</td>
  </tr>
  <tr>
    <td class="tg-0pky">header_item.file_size</td>
    <td class="tg-0pky">0x20</td>
    <td class="tg-0pky">Размер файла</td>
    <td class="tg-0pky">0x10</td>
    <td class="tg-0pky">Размер строки увеличился на 0xF, плюс 0x1 байт выравнивания</td>
  </tr>
  <tr>
    <td class="tg-0pky">header_item.map_off</td>
    <td class="tg-0pky">0x34</td>
    <td class="tg-0pky">оффсет до map</td>
    <td class="tg-0pky">0x10</td>
    <td class="tg-0pky">map находится после массива строк, поэтому оффсет был увеличен, с учетом выравнивания</td>
  </tr>
  <tr>
    <td class="tg-0pky">header_item.data_size</td>
    <td class="tg-0pky">0x68</td>
    <td class="tg-0pky">размер data секции</td>
    <td class="tg-0pky">0x10</td>
    <td class="tg-0pky">Data секция находится после массива строк, поэтому оффсет был увеличен, с учетом выравнивания</td>
  </tr>
  <tr>
    <td class="tg-0pky">map.class_def_item.class_data_off</td>
    <td class="tg-0pky">0xE8</td>
    <td class="tg-0pky">оффсет до данных класса</td>
    <td class="tg-0pky">0xF</td>
    <td class="tg-0pky">Данная структура не требует выравнивания, поэтому значение увеличилось на количество добавленных символов</td>
  </tr>
  <tr>
    <td class="tg-0pky">map_list.debug_info_item</td>
    <td class="tg-0pky">0x114</td>
    <td class="tg-0pky">debug информация</td>
    <td class="tg-0pky">Не важно</td>
    <td class="tg-0pky">Поле хранит данные, необходимые для корректного вывода, при краше. Поле можно игнорировать</td>
  </tr>
</tbody>
</table>

![](/assets/images/infecting_android/img_for_paper/DEX_diff_1.png)

<style type="text/css">
.tg  {border-collapse:collapse;border-spacing:0;}
.tg td{border-color:black;border-style:solid;border-width:1px;font-family:Arial, sans-serif;font-size:14px;
  overflow:hidden;padding:10px 5px;word-break:normal;}
.tg th{border-color:black;border-style:solid;border-width:1px;font-family:Arial, sans-serif;font-size:14px;
  font-weight:normal;overflow:hidden;padding:10px 5px;word-break:normal;}
.tg .tg-0pky{border-color:inherit;text-align:left;vertical-align:top}
</style>
<table class="tg">
<thead>
  <tr>
    <th class="tg-0pky">field</th>
    <th class="tg-0pky">offset</th>
    <th class="tg-0pky">description</th>
    <th class="tg-0pky">diff_count</th>
    <th class="tg-0pky">explanation</th>
  </tr>
</thead>
<tbody>
  <tr>
    <td class="tg-0pky">string_data_item.utf16_size</td>
    <td class="tg-0pky">0x1B3</td>
    <td class="tg-0pky">размер строки</td>
    <td class="tg-0pky">0xF</td>
    <td class="tg-0pky">Строки в DEX хранятся в формате MUTF-8, где один символ занимает 1 байт</td>
  </tr>
</tbody>
</table>

![](/assets/images/infecting_android/img_for_paper/DEX_diff_2.png)

Изменения в конце файла:

<style type="text/css">
.tg  {border-collapse:collapse;border-spacing:0;}
.tg td{border-color:black;border-style:solid;border-width:1px;font-family:Arial, sans-serif;font-size:14px;
  overflow:hidden;padding:10px 5px;word-break:normal;}
.tg th{border-color:black;border-style:solid;border-width:1px;font-family:Arial, sans-serif;font-size:14px;
  font-weight:normal;overflow:hidden;padding:10px 5px;word-break:normal;}
.tg .tg-0pky{border-color:inherit;text-align:left;vertical-align:top}
.tg .tg-0pky{text-align:left;vertical-align:top}
</style>
<table class="tg">
<thead>
  <tr>
    <th class="tg-0pky">field</th>
    <th class="tg-0pky">offset</th>
    <th class="tg-0pky">description</th>
    <th class="tg-0pky">diff_count</th>
    <th class="tg-0pky">explanation</th>
  </tr>
</thead>
<tbody>
  <tr>
    <td class="tg-0pky">map.class_data_item.offset</td>
    <td class="tg-0pky">0x29C</td>
    <td class="tg-0pky">оффсет до class_data_item</td>
    <td class="tg-0pky">0xF</td>
    <td class="tg-0pky">Структура class_data_item следует сразу за массивом строк и не требует выравнивания </td>
  </tr>
  <tr>
    <td class="tg-0pky">map.annotation_set_item.entries.annotation_off_item</td>
    <td class="tg-0pky">0x2A8</td>
    <td class="tg-0pky">оффсет до аннотаций</td>
    <td class="tg-0pky">0x10</td>
    <td class="tg-0pky">Выравнивание учитывается</td>
  </tr>
  <tr>
    <td class="tg-0pky">map.map_list.offset</td>
    <td class="tg-0pky">0x2B4</td>
    <td class="tg-0pky">оффсет до map_list</td>
    <td class="tg-0pky">0x10</td>
    <td class="tg-0pky">Выравнивание учитывается</td>
  </tr>
</tbody>
</table>

![](/assets/images/infecting_android/img_for_paper/DEX_diff_3.png)

Proof-of-Concept код, разработанный для статьи, реализовывает эти модификации в методе `mydex.Patch()`.

## Результаты

Для применения необходимых модификаций, был разработан PoC, который работает по алгоритму:

1. Распаковывание файлов APK
2. Парсинг AndroidManifest.xml
3. Нахождение имени Application класса
4. Патчинг оригинального имени Application на `aaaaaaaa.aaaaaaaaaaaaaaaaaaaa.aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa.aaaaaaaaaaaaaaaaaaaaaa.InjectedApp` в AndroidManifest.xml
5. Патчинг заглушки `z.z.z` на оригинальное имя Application класса
6. Добавление в APK двух DEX (один с InjectedApp Application классом, второй с вредоносными классами)
7. Запаковывание всех файлов в новый APK

## Ограничения нового подхода 

Данная техника не будет работать с приложениями, удовлетворяющие всем условиям одновременно:

1. minSdkVersion <= 20
2. Не используют в зависимостях библиотеку `androidx.multidex:multidex` или `com.android.support:multidex`
3. Запускаются на андроиде версии меньше, чем Android 5.0 (API level 21) 

либо одному условию:

1. У приложения манифесте существует опция `android:extractNativeLibs=false`. В этом случае, при установке apk вы будете получать ошибку `INSTALL_FAILED_INVALID_APK: Failed to extract native libraries, res=-2`


Тем самым предполагается, что приложение имеет один DEX файл. Ограничение применимо из-за того, что версии андроида до Android 5.0 (API level 21) используют виртуальную машину Dalvik, для запуска кода. По умолчанию, Dalvik воспринимает только один DEX файл в APK. Чтобы обойти это ограничение, необходимо использовать вышеприведенные библиотеки. Версии андроида после Android 5.0 (API level 21), вместо Dalvik, используют систему ART, которая нативно поддерживает несколько DEX файлов в приложении, так как при установке приложения она прекомпилирует все DEX в один .oat файл. Подробности написаны в [официальной документации](https://developer.android.com/studio/build/multidex).

## Дальнейшие улучшения PoC

1. Если у приложения нет своего Application класс, то необходимо добавлять `InjectedApp` в AndroidManifest.xml
2. Добавление в AndroidManifest.xml своих тегов
3. Подписание APK
4. Избавление от декодирования AndroidManifest.xml
5. Обход ошибки при установке приложений, с опцией манифест `android:extractNativeLibs=false`

## FAQ

Q: Почему бы не использовать в полном имени InjectedApp символы подчеркивания, тем самым оно практически гарантировано будет по алфавиту выше любого имени Application класса целевого приложения?

A: Технически это возможно, но возникнут проблемы в Android 5 и будет следующая ошибка:

```
D/AndroidRuntime( 3891): Calling main entry com.android.commands.pm.Pm
D/DefContainer( 3414): Copying /mnt/shared/App/20200629234847850.apk to base.apk
W/PackageManager( 1802): Failed parse during installPackageLI
W/PackageManager( 1802): android.content.pm.PackageParser$PackageParserException: /data/app/vmdl1642407162.tmp/base.apk (at Binary XML file line #48): Bad class name ________.__________._0000000000000000000000000000000000000000000000000000000000000000.InjectedApp in package XXXXXXXXXXXXXXXXXXXXXX
W/PackageManager( 1802):        at android.content.pm.PackageParser.parseBaseApk(PackageParser.java:885)
W/PackageManager( 1802):        at android.content.pm.PackageParser.parseClusterPackage(PackageParser.java:790)
W/PackageManager( 1802):        at android.content.pm.PackageParser.parsePackage(PackageParser.java:754)
W/PackageManager( 1802):        at com.android.server.pm.PackageManagerService.installPackageLI(PackageManagerService.java:10816)
W/PackageManager( 1802):        at com.android.server.pm.PackageManagerService.access$2300(PackageManagerService.java:236)
W/PackageManager( 1802):        at com.android.server.pm.PackageManagerService$6.run(PackageManagerService.java:8888)
W/PackageManager( 1802):        at android.os.Handler.handleCallback(Handler.java:739)
W/PackageManager( 1802):        at android.os.Handler.dispatchMessage(Handler.java:95)
W/PackageManager( 1802):        at android.os.Looper.loop(Looper.java:135)
W/PackageManager( 1802):        at android.os.HandlerThread.run(HandlerThread.java:61)
W/PackageManager( 1802):        at com.android.server.ServiceThread.run(ServiceThread.java:46)
```

Q: Почему бы вместо внедрения своего Application класса, не внедрять свой Activity и прописывать его в манифесте, вместо основного, ведь оно также стартует самым первым? Да, при таком способе payload выполнится чуть позже, но это не критично.

A: В этом подходе есть две проблемы. Первая - существуют приложения, которые используют в манифесте очень много тегов [activity-alias](https://developer.android.com/guide/topics/manifest/activity-alias-element), которые ссылаются на имя основного активити. В этом случае нам придется патчить не одну строку в манифесте, а несколько. Также это затрудняет парсинг и нахождение имени нужного Activity. Вторая - основной Activity запускается в главном UI потоке, что накладывает некоторые ограничения на вредоносный код.

Q: Но ведь в Application классе нельзя использовать сервисы. Какой может быть вредоносный код без сервисов?

A: Во-первых, это ограничение введено в версии андроида, начиная с API 25. Во-вторых, это ограничение касается андроид приложений в целом, а не конкретно Application класса. В третьих, сервисы использовать можно, но не обычные, а foreground. 

Q: Ваш PoC не работает

A: В этом случае удостоверьтесь, что:
   1. Оригинальное приложение работает
   2. Все пути к файлам в PoC корректны
   3. В apkinfector.log нету ничего необычного
   4. Имя оригинального Application класса в пропатченном InjectedApp.dex действительно находится на своем месте 
   5. Целевое приложение использует свой Application класс. Иначе, неработоспособность PoC - предсказуема

Если ничего не помогло, то попробуйте поиграть с параметром `--min-api`, когда компилируете классы.
Если ничего не помогло, то создайте issue на github.

Q: Почему для заражения был выбран конструктор Application, а не метод OnCreate()? 

A: Дело в том, что существуют приложения, у которых Application класс содержит метод OnCreate() с модификатором final. Если вы подложите свой Application с OnCreate(), то андроид выдаст ошибку:

```
06-28 07:27:59.770  2153  4539 I ActivityManager: Start proc 6787:xxxxxxxxx/u0a46 for activity xxxxxxxxx/.Main
06-28 07:27:59.813  6787  6787 I art     : Rejecting re-init on previously-failed class java.lang.Class<InjectedApp>:
 java.lang.LinkageError: Method void InjectedApp.onCreate() overrides final method in class LX/001; 
(declaration of 'InjectedApp' appears in /data/app/xxxxxxxxx-1/base.apk:classes2.dex)
```

Причины ошибки [тут](https://android.googlesource.com/platform/art/+/refs/heads/master/runtime/class_linker.cc#6640)

```
if (super_method->IsFinal()) {
          ThrowLinkageError(klass.Get(), "Method %s overrides final method in class %s",
                            virtual_method->PrettyMethod().c_str(),
                            super_method->GetDeclaringClassDescriptor());
          return false;
        }
``` 

Андроид увидит, что super method - final и выдаст ошибку.

В Java, если вы не создали никакого конструктора, то компилятор создаст его за вас (без параметров). Если же вы создали конструктор с параметрами, то конструктор без параметров автоматически не создается. Так как мы вызываем конструктор без параметров, то вы можете подумать, что возникнет проблема, если app class целевого приложения содержит констурктор с параметрами. Но нет, именно для Application классов, андроид требует чтобы был дефолтный констурктор. Иначе вы получаете такую ошибку

```
06-28 08:51:54.647  8343  8343 D AndroidRuntime: Shutting down VM
06-28 08:51:54.647  8343  8343 E AndroidRuntime: FATAL EXCEPTION: main
06-28 08:51:54.647  8343  8343 E AndroidRuntime: Process: xxxxxxxxx, PID: 8343
06-28 08:51:54.647  8343  8343 E AndroidRuntime: java.lang.RuntimeException: Unable to instantiate application xxxxxxxxx.AppShell: java.lang.InstantiationException: java.lang.Class<xxxxxxxxx.AppShell> has no zero argument constructor
06-28 08:51:54.647  8343  8343 E AndroidRuntime:        at android.app.LoadedApk.makeApplication(LoadedApk.java:802)
```
