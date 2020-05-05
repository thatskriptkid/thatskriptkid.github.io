---
layout: post
title: Как дебажить библиотеки андроид приложений или Решаем "OWASP UnCrackable App for Android Level 3" или Crackme from HELL
tags: [android, crackme]
category: [ru]
---

## Intro

[Первая часть]({% post_url ru/2019-12-12-owasp-crackme-1 %})

[Вторая часть]({% post_url ru/2019-12-17-owasp-crackme-2 %})

Советую прочитать предыдущие части, так как многие моменты описанные там, здесь описываться не будут.

Tools used: [apktool](https://ibotpeaches.github.io/Apktool/), [IDA PRO](https://rada.re/n/)

Продолжаем решать задания [от OWASP](https://github.com/OWASP/owasp-mstg/tree/master/Crackmes). Эти задания даются в качестве демонстративных примеров в [OWASP Mobile Security Testing Guide](https://www.owasp.org/index.php/OWASP_Mobile_Security_Testing_Guide). Я не читал этот гайд и буду пытаться сделать задания, опираясь только на свои обрывки знаний об андроиде. Скорее всего есть более быстрые и короткие пути решения. Решение ниже просто является моим.

Третий крякми достаточно интересный, в плане демонстрации уже более сложного подхода к защите секретной строки.

## UnCrackable App for Android Level 3

Пугающее описание задания третьего уровня

```
The crackme from hell!
Objective: A secret string is hidden somewhere in this app. Find a way to extract it.
```

Скачиваем apk, закидываем на эмулятор, запускаем

![](/assets/images/ru/owasp-3/1.png)

В этот раз нас встречают новым сообщением, к которому добавился "tampering" (переведем, как "вмешательство"). Открываем исходники `MainActivity` и смотрим, не появились ли новые проверки? В декодированом apktool'ом виде, кода на smali получается настолько много, что для наглядности будем рассматривать декомпилированный вариант. Я декомпилировал [тут](http://www.javadecompilers.com/apk).

На входе нас встречает вот такая функция

```java
public void onCreate(Bundle bundle) {
        verifyLibs();
```

Откроем ее и разберем по кускам

```java
private void verifyLibs() {
        ...
        this.crc = new HashMap();
        this.crc.put("armeabi-v7a", Long.valueOf(Long.parseLong(getResources().getString(R.string.armeabi_v7a))));
        this.crc.put("arm64-v8a", Long.valueOf(Long.parseLong(getResources().getString(R.string.arm64_v8a))));
        this.crc.put("x86", Long.valueOf(Long.parseLong(getResources().getString(R.string.x86))));
        this.crc.put("x86_64", Long.valueOf(Long.parseLong(getResources().getString(R.string.x86_64))));
        ...
```

Данный код, достает из файла `strings.xml` (лежит в ресурсах) значение `armeabi_v7a`.

```java
Long.valueOf(Long.parseLong(getResources().getString(R.string.armeabi_v7a)))
```

![](/assets/images/ru/owasp-3/2.png)

Аналогично и с остальными. Имена: `armeabi_v7a`, `arm64_v8a`, `x86`, `x86_64` - являются идентификаторами архитектур. 

<details> 
  <summary>Про нативные библиотеки</summary>
   Если андроид приложение использует нативные библиотеки и хочет работать на разных системах (старых/новых телефонах/планшетах и т.д.), оно должно содержать в себе библиотеки скомпилированные под каждую из архитектур. Нативные библиотеки - это библиотеки, которые содержат код, написанный на С/С++. В реальных приложениях, разработчики обычно запихивают туда:

1. Работу с SSL соединением, чтобы предотвратить прослушивание трафика (например Instagram)
2. Код, который проще написать на С/С++
3. Код, который должен работать быстро
4. Код, логику которого надо скрыть от реверсера, так как читать ARM asm больнее, чем java 
</details>

Значениями являются, судя по названию HashMap, контрольные суммы [CRC](https://ru.wikipedia.org/wiki/%D0%A6%D0%B8%D0%BA%D0%BB%D0%B8%D1%87%D0%B5%D1%81%D0%BA%D0%B8%D0%B9_%D0%B8%D0%B7%D0%B1%D1%8B%D1%82%D0%BE%D1%87%D0%BD%D1%8B%D0%B9_%D0%BA%D0%BE%D0%B4). То есть, мы положили в структуру вида key-value название архитектуры и контрольную сумму.

Следующий кусок кода этой функции

```java

            ZipFile zipFile = new ZipFile(getPackageCodePath());
            Iterator it = this.crc.entrySet().iterator();
            while (true) {
                str = ", supposed to be ";
                str2 = "] = ";
                str3 = "CRC[";
                if (!it.hasNext()) {
                    break;
                }
                Entry entry = (Entry) it.next();
                StringBuilder sb = new StringBuilder();
                sb.append("lib/");
                sb.append((String) entry.getKey());
                sb.append("/libfoo.so");
                String sb2 = sb.toString();
                ZipEntry entry2 = zipFile.getEntry(sb2);
                StringBuilder sb3 = new StringBuilder();
                sb3.append(str3);
                sb3.append(sb2);
                sb3.append(str2);
                sb3.append(entry2.getCrc());
                Log.v(str4, sb3.toString());
                if (entry2.getCrc() != ((Long) entry.getValue()).longValue()) {
                    tampered = 31337;
                    StringBuilder sb4 = new StringBuilder();
                    sb4.append(sb2);
                    sb4.append(": Invalid checksum = ");
                    sb4.append(entry2.getCrc());
                    sb4.append(str);
                    sb4.append(entry.getValue());
                    Log.v(str4, sb4.toString());
                }
            }
```

В этом коде, мы получаем путь к apk нашего приложения, проходим по всем библиотекам (для каждой архитектуры) в папке lib, считаем ее контрольную сумму и сравниваем со значениями в нашей предыдущей структуре. Очевидно, что это проверка на то, не изменили ли мы библиотеки приложения (вот откуда и взялось слово "tampering" в сообщении). Если контрольные суммы не совпадают, нам говорят "Invalid checksum".

Смотрим следующий кусок кода

```java
            String str5 = "classes.dex";
            ZipEntry entry3 = zipFile.getEntry(str5);
            StringBuilder sb5 = new StringBuilder();
            sb5.append(str3);
            sb5.append(str5);
            sb5.append(str2);
            sb5.append(entry3.getCrc());
            Log.v(str4, sb5.toString());
            if (entry3.getCrc() != baz()) {
                tampered = 31337;
                StringBuilder sb6 = new StringBuilder();
                sb6.append(str5);
                sb6.append(": crc = ");
                sb6.append(entry3.getCrc());
                sb6.append(str);
                sb6.append(baz());
                Log.v(str4, sb6.toString());
            }
```

Здесь идет проверка контрольной суммы файла "classes.dex" с результатом функции `baz()`, тело которой находится в библиотеки, судя по модификатору native

```java
private native long baz();
```

<details> 
  <summary>Про classes.dex</summary>

  Если вы возьмете apk любого приложения и просто распакуете zip'ом, то увидите наличие этого файла, а может даже нескольких.

![](/assets/images/ru/owasp-3/3.png)

Исходный код андроид приложения компилируется, с помощью компилятора javac и превращается в java байткод. Такой же обычный байткод, который мы получаем при компиляции любой java программы. Далее, он еще раз компилируется с помощью DX компилятора и в итоге мы получаем цельный DEX файл. Этот файл содержит код, который непосредственно исполняется виртуальной машиной андроида. То есть, любое андроид приложение, содержит DEX. DEX - бинарный файл и содержит инструкции для dalvik.
Этот файл содержит все классы приложения, в скомпилированном виде. Нарисовал вот такую картинку, для понимания

![](/assets/images/ru/owasp-3/4.png)

Если приложение содержит больше 65к методов, то автоматически создается еще один dex файл. Некоторые приложения содержат с десяток таких файлов, которые просто именуются classes2.dex, classes3.dex, ...
</details>

Посмотрим на код функции `baz()`. Смотрим код этой функции таким же способом, как и во [второй части](https://github.com/thatskriptkid/OrderOfSixAngles/wiki/%D0%A0%D0%B5%D1%88%D0%B0%D0%B5%D0%BC-%22OWASP-UnCrackable-App-for-Android-Level-2%22,-%D1%81-%D0%BF%D0%BE%D0%BC%D0%BE%D1%89%D1%8C%D1%8E-FRIDA-%D0%B8-Binary-Ninja!). Вкратце, нам надо взять so файл в папке lib и отправить в декомпилятор.

![](/assets/images/ru/owasp-3/5.png)

Это код x86 архитектуры, а значит возвращаемое значение функции помещается в регистр `EAX`. Значит контрольная сумма файла `classes.dex` сравнивается со значением `0x18110e3`. Это и есть наши проверки на tempering.

С функцией `verifyLibs()` завершили, продолжим

```java
new AsyncTask<Void, String, String>() {
            /* access modifiers changed from: protected */
            public String doInBackground(Void... voidArr) {
                while (!Debug.isDebuggerConnected()) {
                    SystemClock.sleep(100);
                }
                return null;
            }

            /* access modifiers changed from: protected */
            public void onPostExecute(String str) {
                MainActivity.this.showDialog("Debugger detected!");
                System.exit(0);
            }
        }.execute(new Void[]{null, null, null});

        if (RootDetection.checkRoot1() || RootDetection.checkRoot2() || RootDetection.checkRoot3() || IntegrityCheck.isDebuggable(getApplicationContext()) || tampered != 0) {
            showDialog("Rooting or tampering detected.");
        }
```

Здесь вы можете увидеть антидебагинг технику (`Debug.isDebuggerConnected()`) и чек на рут (`RootDetection.*`). Выпиливание проверок рассматривалось в [первой части](https://github.com/thatskriptkid/OrderOfSixAngles/wiki/%D0%A0%D0%B5%D1%88%D0%B0%D0%B5%D0%BC-%22OWASP-UnCrackable-App-for-Android-Level-1%22-%D0%B8%D0%BB%D0%B8-%D1%83%D1%87%D0%B8%D0%BC-smali), а фридой [тут](https://github.com/thatskriptkid/OrderOfSixAngles/wiki/%D0%A0%D0%B5%D1%88%D0%B0%D0%B5%D0%BC-%22OWASP-UnCrackable-App-for-Android-Level-2%22,-%D1%81-%D0%BF%D0%BE%D0%BC%D0%BE%D1%89%D1%8C%D1%8E-FRIDA-%D0%B8-Binary-Ninja!).

Так как мы вносим изменения в приложение, функцию `verifyLibs()` тоже выпиливаем из кода.

Изучаем код дальше

```java
public void onCreate(Bundle bundle) {
        verifyLibs();
        init(xorkey.getBytes());
```

Теперь нас интересует функция `init`, которая снова `native`, и принимает некий ключ

```java
private static final String xorkey = "pizzapizzapizzapizzapizz";
private native void init(byte[] bArr);
```

Откроем код этой функции в любом дизассемблере (я использовал [Binary Ninja](https://binary.ninja/)), который находится в единственной библиотеке `libfoo.so`, в папке `lib` приложения

![](/assets/images/ru/owasp-3/6.png)

Я сел поудобнее и начал смотреть на этот код, ожидая просветления. Небольшой спойлер - оно не пришло. Обратите внимание, в коде есть вызовы функций с динамическим адресом (выделено красным), а значит статический анализ тут бесполезен, надо дебажить в рантайме. Далее будет мини туториал, как дебажить код в андроид библиотеках, с помощью IDA Pro.

## Дебажим библиотеки 

Иногда появляется потребность пройтись дебагером по коду в библиотеке. Для этого нам потребуется физический андроид девайс, рутованый на arm32 (armeabi-v7a). Эти требования являются необязательными, но желательными.

 _Отдельный физический девайс_ - так как встроенный эмулятор в Android Studio очень плохо работает на ARM32, а остальные мне просто лень скачивать. 

 _Рутованый_ - так как надо будет запустить дебаг сервер от рута. 

 _arm32_ - потому что он проще, чем arm64.

Почему мы не рассматриваем x86 или x86_64? Потому что данная статья в том числе и образовательная и не все приложения имеют библиотеки под данные архитектуры, а значит наш метод можно распространять на практически любое реальное андроид приложение, не только crackme.

Приложение находится на телефоне, дебажить мы будем со своего компа. Значит мы будем использовать remote debugging. Remote debugging - это когда мы подключаемся дебагером удаленно. Чтобы этого достичь, на устройстве, где мы будем дебажить, надо запустить дебаг-сервер. Дебагер на компе будет выступать в роли клиента. В папке IDA PRO, есть папка dbgsrv, копируем оттуда файл `android_server` на телефон и запускаем

```
adb push ./android_server /data/local/tmp
adb root
adb shell
chmod 755 /data/local/tmp/android_server
/data/local/tmp/android_server
IDA Android 32-bit remote debug server(ST) v1.22. Hex-Rays (c) 2004-2017
Listening on 0.0.0.0:23946...
```

Если вы все сделали правильно, то вывод будет говорить о том, что дебаг сервер ожидает подключения. Если у вас не получается:

1. Вы скорее всего скопировали не тот дебаг сервер
2. Вы не выставили файлу правильные права
3. Ваш телефон не arm32

Теперь настраиваем port forwarding, одной командой adb, с указанием порта нашего дебаг сервера

```
adb forward tcp:23946 tcp:23946
```

Следующий шаг - включение дебаг режима в самом приложении. Это необходимо для того, чтобы мы могли подключиться идой. Распаковываем apktool'ом и добавляем в манифест

```xml
android:debuggable="true"
```

Собираем приложение и подписываем.

<details> 
  <summary>Как подписать?</summary>

   Устанавливаем JDK (Java development kit), добавляем его в PATH. Там лежит jarsigner и keytool.

Генерируем keystore:

```
keytool -genkey -v -keystore my-release-key.keystore -alias alias_name -keyalg RSA -keysize 2048 -validity 10000
```

Подписываем приложение 

```
jarsigner -verbose -sigalg SHA1withRSA -digestalg SHA1 -keystore my-release-key.keystore my_application.apk alias_name
```
</details>

Устанавливаем приложение на телефон. Теперь, что его дебажить, активируем опцию разработчика и в меню "Отслеживаемые приложения" выбираем наше

![](/assets/images/ru/owasp-3/7.png)

Включаем "Подождать отладчик", чтобы приложение не стартовало без нашего дебагера

![](/assets/images/ru/owasp-3/8.png)

Всё, теперь наш телефон полностью готов.

## Дебажим с двух концов

Дебажить мы будем, с помощью IDA PRO. Ида не может одновременно дебажить приложение и его библиотеки. Поэтому мы будем дебажить сразу с двух запущенных ИД. В первой иде мы будем дебажить java классы, во второй библиотеку. Открываем наш apk и выбираем classes.dex. 

![](/assets/images/ru/owasp-3/9.png)
![](/assets/images/ru/owasp-3/10.png)

classes.dex содержит все классы приложения, поэтому мы видим функции всех классов. Нас интересует `MainActivity.OnCreate()`

![](/assets/images/ru/owasp-3/11.png)

Ставим брейкпоинт куда-нибудь в начало. Открываем меню Debugger - Debugger options - Set specific options. Здесь нам надо указать package name и активити. Можно даже это сделать из манифеста

![](/assets/images/ru/owasp-3/12.png)

Все, мы настроили первый инстанс иды. На нем мы будем стартовать приложение, подцепляться дебаггером, и дебажить `MainActivity.Oncreate()`

Запускаем еще одну иду. В этой иде мы будем дебажить библиотеку, поэтому открываем файл libfoo.so в папке lib приложения. Нас интересует функция `init()`. Ставим брейкпоинт на ее начало. 

![](/assets/images/ru/owasp-3/13.png)

В опциях выбираем "Remote ARM Linux/Android debugger", а в Debugger > Process Options

![](/assets/images/ru/owasp-3/14.png)

localhost указали, так как мы уже пробросили порты. Переход в нашу первую иду и нажимаем Start. Ида стартанет наше приложение и вы должны в начале видеть вот такое окно на телефоне

![](/assets/images/ru/owasp-3/15.png)

После небольшой паузы, вы встанете на ранее установленный брейкпоинт.

![](/assets/images/ru/owasp-3/16.png)

Теперь ничего не закрывая и не трогая, открываем вторую иду и выбираем Debugger - Attach to process. Откроется окно со списком процессов на телефоне, выбираем наш crackme

![](/assets/images/ru/owasp-3/17.png)

Теперь в первой иде жмем Continue (F9). После этого, во второй иде, мы встанем на брейкпоинт в функции `init()`!

![](/assets/images/ru/owasp-3/18.png)

_Функция содержит код, который определяет хуки Xposed и FRIDA. Мы не используем их в решении, поэтому нам это не интересно._

Суть этой функции сводится к единственному вызову `strncpy()`. Она копирует xor ключ, который мы видели ранее, в память.

![](/assets/images/ru/owasp-3/19.png)

Ключ лежит по адресу, который находится в регистре R1. 

![](/assets/images/ru/owasp-3/20.png)

С ней разобрались. Дальше можно нажать Continue и увидеть уже основной экран приложения

![](/assets/images/ru/owasp-3/21.png)

Здесь от нас ожидают секретную строку. Давайте посмотрим на код функции, который ее проверяет

```java
public void verify(View view) {
        String obj = ((EditText) findViewById(R.id.edit_text)).getText().toString();
        AlertDialog create = new Builder(this).create();
        if (this.check.check_code(obj)) {
            create.setTitle("Success!");
            create.setMessage("This is the correct secret.");
        } else {
            create.setTitle("Nope...");
            create.setMessage("That's not it. Try again.");
        }
        create.setButton(-3, "OK", new OnClickListener() {
            public void onClick(DialogInterface dialogInterface, int i) {
                dialogInterface.dismiss();
            }
        });
        create.show();
    }
```

Проверка происходит в функции `this.check.check_code(obj)`. Откроем ее

```java
public class CodeCheck {
    private static final String TAG = "CodeCheck";

    private native boolean bar(byte[] bArr);

    public boolean check_code(String str) {
        return bar(str.getBytes());
    }
}
```

Ее код тоже находится в библиотеки, в функции `bar()`. Для нас уже не проблема дебажить native функции, поэтому во второй иде (там где мы открывали быблиотеку), ставим на нее брейкпоинт

![](/assets/images/ru/owasp-3/22.png)

Вводим строку "AAAAAAAAAAAAAA" и нажимаем verify. 

![](/assets/images/ru/owasp-3/23.png)

Срабатывает брейкпоинт внутри функции `bar()`. В начале идет получение длины введеной строки JNI функцией `GetArrayLength()`

![](/assets/images/ru/owasp-3/24.png)

И сравнение ее с константой 0х18.

![](/assets/images/ru/owasp-3/25.png)

Поэтому мы видим такое окно

![](/assets/images/ru/owasp-3/26.png)

Теперь давайте введем строку длиной 24. Проверку мы проходим и далее нас встречает вот такой код.

![](/assets/images/ru/owasp-3/27.png)

Можно догадаться, что это цикл. `R0` - это номер итерации, инструкция `ADDS` увеличивает ее на 1 каждый шаг. Рассмотрим каждую инструкцию в теле цикла подробнее

```
LDRB.W          R2, [R10,R0]
```

_Что делают инструкции ARM я смотрел [тут](http://infocenter.arm.com/help/index.jsp)_.

LDRB (Load register byte) - кладет 1 байт по адресу `[R10]` в `R2`. По адресу `[R10]` у нас лежат какие-то однобайтные константы

![](/assets/images/ru/owasp-3/28.png)

```
LDRB            R1, [R6,R0]
```

Кладем один байт по адресу `[R6]` в `R1`. В `[R6]` у нас находится наша строка

![](/assets/images/ru/owasp-3/29.png)

```
LDRB            R3, [R3,#4]
```

В `[R3]` лежит строка `pizzapizzapizzapizzapizz`, которую мы скопировали в память, в функции `init()`

![](/assets/images/ru/owasp-3/30.png)

В итоге:

`R0` - счетчик цикла

`R1` - наш ввод

`R2` - константы 

`R3` - pizzapizzapizzapizzapizz

Далее идут

```
EORS            R2, R3
CMP             R1, R2
```

EORS - это просто XOR. Символы строки `pizzapizzapizzapizzapizz` ксорятся с константами в `[R2]` и сравниваются (`CMP`) с нашим вводом. Это и есть вся проверка. Нам осталось воспроизвести алгоритм на питоне

```python
str = 'pizzapizzapizzapizzapizz'
consts = [29, 8, 17, 19, 15, 23, 73, 21, 13, 0, 3, 25, 90, 29, 19, 21, 8, 14, 90, 0, 23, 8, 19, 20]
res = ''

for x in range(0, len(str)):
    res += chr(ord(str[x]) ^ consts[x])

print(res)
```

В итоге получим строку "making owasp great again"

![](/assets/images/ru/owasp-3/31.png)

## Итог

В заключении хочу заметить, что мы не вмешивались в код библиотеки, хотя проверка целостности была. После написания этой статьи, я прочитал решения других людей и увидел, что оказывается приложение проверяет наличие фреймворков FRIDA и Xposed. Но нас это не коснулось, мы их попросту не использовали. В другом решении, человек просто запатчил библиотеку так, чтобы она выдавала все строки в готовом виде. Но разве не интереснее все сделать в рантайме? Разве не интереснее несколько часов долбится в стену, а потом понять, что надо дебажить сразу с двух ИД одновременно? Разве может быть что-то приятнее, чем дебажить ARM32 код, абсолютно его незная? =)

В любом случае, всем спасибо за внимание.
