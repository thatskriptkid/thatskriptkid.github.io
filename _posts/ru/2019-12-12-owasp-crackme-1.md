---
layout: post
title: Решаем "OWASP UnCrackable App for Android Level 1" или учим smali
tags: [android, crackme]
category: [ru]
---

Я расскажу о своем решении заданий на реверс андроид приложений, ([от OWASP](https://github.com/OWASP/owasp-mstg/tree/master/Crackmes)). Эти задания даются в качестве демонстративных примеров в [OWASP Mobile Security Testing Guide](https://www.owasp.org/index.php/OWASP_Mobile_Security_Testing_Guide). Я не читал этот гайд и буду пытаться сделать задания, опираясь только на свои обрывки знаний об андроиде. Скорей всего есть более быстрые и короткие пути решения, эти решения просто являются моими.

## UnCrackable App for Android Level 1

Первый уровень встречает нас таким описанием

<pre>
This app holds a secret inside. Can you find it?
Objective: A secret string is hidden somewhere in this app. Find a way to extract it.
</pre>

Скачиваем apk, устанавливаем на android эмулятор, запускаем. Видим такое окно

![](/assets/images/ru/owasp-1/1.png)

На фоне видем форму, в которую по всей видимости нам надо ввести secret. Но форма не доступна, приложение определило, что было запущено на эмуляторе (рутованый) и закрылось. Давайте для начала разберемся, откуда берется это окно и попробуем убрать его. Для этого нам нужно посмотреть исходный код apk и изменить его. Есть два способа получения исходного кода приложения - декомпилирование и декодирование в байткод. Декомпиляция даст нам java классы, которые удобнее читать. Но внести изменения в них и собрать заново в apk мы с большей вероятностью не сможем. Поэтому мы будем использовать утилиту apktool, чтобы получить исходный код в виде smali. Его сложнее читать, но при изменении приложение пересоберается безболезненно. Декодируем.

![](/assets/images/ru/owasp-1/2.png)

Теперь нам надо открыть исходник Activity, который открывается при старте. Для этого откроем файл AndroidManifest.xml в папке base.

```xml
<?xml version="1.0" encoding="utf-8" standalone="no"?><manifest xmlns:android="http://schemas.android.com/apk/res/android" package="owasp.mstg.uncrackable1">
    <application android:allowBackup="true" android:icon="@mipmap/ic_launcher" android:label="@string/app_name" android:theme="@style/AppTheme">
        <activity android:label="@string/app_name" android:name="sg.vantagepoint.uncrackable1.MainActivity">
            <intent-filter>
                <action android:name="android.intent.action.MAIN"/>
                <category android:name="android.intent.category.LAUNCHER"/>
            </intent-filter>
        </activity>
    </application>
</manifest>
```

Находим имя класса `sg.vantagepoint.uncrackable1.MainActivity`. По этому же пути в папке `base/smali` лежит исходник, открываем его и смотрим метод `OnCreate()`, так как именно он запускается самым первым. 

## Немного о smali

Андроид приложения компилируются в байткод, который исполняется виртуальной машиной Dalvik. Сам байткод тяжело читаем, поэтому его удобочитаемая для человека форма, называется smali. smali - это аналог языка ассемблера, но для андроида. Он на самом деле достаточно просто. Чтобы дальнейший текст был более понятен, изучим основные команды (полная информация вы найдете [в официальной документации](https://source.android.com/devices/tech/dalvik/dalvik-bytecode):

**const-string v0, "test"** - кладем строку "test" в переменную `v0` (правильней говорить "в регистр", но опустим это)

**invoke-direct** - вызов нестатичного метода

**invoke-static** - вызов статичного метода

**invoke-virtual** - вызов метода, который не является private, static или final

**if-nez** - "if-not-equal-zero", условие "если меньше или равно нулю" (аналогично применимо к if-eqz, if-gez, if-lez и т.д.)

**move-result v0, move-result-object v0, ...** - поместить результат выполнение функции в переменную v0

Продолжим. В теле метода `OnCreate()` видим такой код

```java
    :cond_0 //метка для перехода (как label в Си для goto)
    const-string v0, "Root detected!" // помещаем в переменную v0 строку
    
    // invoke-direct означаем вызов нестатиченого метода. v0 - это передаваемый параметр
    invoke-direct {p0, v0}, Lsg/vantagepoint/uncrackable1/MainActivity;->a(Ljava/lang/String;)V
```

Судя по тексту, метод `a()` выполняется, когда приложение обнаружило рут. Если мы посмотрим на него

```java
.method private a(Ljava/lang/String;)V
    .locals 3

    new-instance v0, Landroid/app/AlertDialog$Builder;

    invoke-direct {v0, p0}, Landroid/app/AlertDialog$Builder;-><init>(Landroid/content/Context;)V

    invoke-virtual {v0}, Landroid/app/AlertDialog$Builder;->create()Landroid/app/AlertDialog;

    move-result-object v0

    invoke-virtual {v0, p1}, Landroid/app/AlertDialog;->setTitle(Ljava/lang/CharSequence;)V

    const-string p1, "This is unacceptable. The app is now going to exit."

    invoke-virtual {v0, p1}, Landroid/app/AlertDialog;->setMessage(Ljava/lang/CharSequence;)V

    const-string p1, "OK"

    new-instance v1, Lsg/vantagepoint/uncrackable1/MainActivity$1;

    invoke-direct {v1, p0}, Lsg/vantagepoint/uncrackable1/MainActivity$1;-><init>(Lsg/vantagepoint/uncrackable1/MainActivity;)V

    const/4 v2, -0x3

    invoke-virtual {v0, v2, p1, v1}, Landroid/app/AlertDialog;->setButton(ILjava/lang/CharSequence;Landroid/content/DialogInterface$OnClickListener;)V

    const/4 p1, 0x0

    invoke-virtual {v0, p1}, Landroid/app/AlertDialog;->setCancelable(Z)V

    invoke-virtual {v0}, Landroid/app/AlertDialog;->show()V

    return-void
.end method
```

то увидим, что в нем создается `AlertDialog` с надписью "_This is unacceptable. The app is now going to exit._" и кнопкой. Это совпадает, с тем что мы видели на старте. А это значит, что переход по метке `:cond_0` выводит нам это окно. Давайте посмотрим, что нас приводит к нему, возвращаемся в метод `OnCreate()` и смотрим код, с нужным условием.

```java
    //invoke-static вызов статичного метода
    invoke-static {}, Lsg/vantagepoint/a/c;->a()Z 

    move-result v0
    // if-nez - "if not equal zero" - если результат работы функции не равен 0, то мы кричим, что обнаружили рут
    // аналогично с функцией b() ниже
    if-nez v0, :cond_0

    invoke-static {}, Lsg/vantagepoint/a/c;->b()Z

    move-result v0

    if-nez v0, :cond_0
```

Очевидно, что функции `sg/vantagepoint/a/c;->a()`, `sg/vantagepoint/a/c;->b()` и если почитать остальной код метода - `Lsg/vantagepoint/a/c;->c()` - чекают рут. Они находятся в классе `sg.vantagepoint.a.c`, посмотрим на них

Код `a()`

```java

    ...

    new-instance v5, Ljava/io/File;

    const-string v6, "su"

    invoke-direct {v5, v4, v6}, Ljava/io/File;-><init>(Ljava/lang/String;Ljava/lang/String;)V

    invoke-virtual {v5}, Ljava/io/File;->exists()Z
    ...
```
Здесь проверяется наличие файла "su". Это пример кода проверки рута, на рутованом телефоне всегда будет находиться бинарник su. Поиск осуществляется в getenv("PATH")

```java

    ...

    const-string v0, "PATH"

    invoke-static {v0}, Ljava/lang/System;->getenv(Ljava/lang/String;)Ljava/lang/String;

    ...
```

Посмотрим на второй метод `sg/vantagepoint/a/c;->b()`

```java

    ...

    sget-object v0, Landroid/os/Build;->TAGS:Ljava/lang/String;

    if-eqz v0, :cond_0

    const-string v1, "test-keys"

    invoke-virtual {v0, v1}, Ljava/lang/String;->contains(Ljava/lang/CharSequence;)Z

    ...
```

Здесь используется другой способ обнаружения рута, а именно в переменной Build.TAGS (лежит в /system/build.prop, на исходники можно посмотреть [тут](https://android.googlesource.com/platform/frameworks/base/+/refs/heads/master/core/java/android/os/Build.java)) идет поиск строки test-keys. Дело в том, что когда ядро подписывается официальным ключом, то в Build.TAGS будет лежать строка "Release-Keys". "test-key" означает, что ядро было подписано тестовым публичным ключом [AOSP](https://source.android.com/). 

Остался последний метод `c()`

```java
    invoke-static {}, Lsg/vantagepoint/a/c;->c()Z

    move-result v0

    if-eqz v0, :cond_1
```

Он вот такой

```java
.method public static c()Z
    ...

    const-string v0, "/system/app/Superuser.apk"

    const-string v1, "/system/xbin/daemonsu"

    const-string v2, "/system/etc/init.d/99SuperSUDaemon"

    const-string v3, "/system/bin/.ext/.su"

    const-string v4, "/system/etc/.has_su_daemon"

    const-string v5, "/system/etc/.installed_su_daemon"

    const-string v6, "/dev/com.koushikdutta.superuser.daemon/"

    ...

    invoke-direct {v5, v4}, Ljava/io/File;-><init>(Ljava/lang/String;)V

    invoke-virtual {v5}, Ljava/io/File;->exists()Z

   ...
.end method
```

Здесь, мы по заранее захардкоженым пустям, ищем бинарники, которые используются эксплоитами, для рутования.

Все это мы проделали, чтобы увидеть техники рутования и немного изучить smali. Для решения задачи, так глубоко рабираться в этом, не так необходимо. Мы можем просто все проверки выпилить из кода. После выпиливания всех проверок на рут, наш метод `OnCreate()` выглядит так

```java
# virtual methods
.method protected onCreate(Landroid/os/Bundle;)V
    .locals 1

    invoke-super {p0, p1}, Landroid/app/Activity;->onCreate(Landroid/os/Bundle;)V

    const/high16 p1, 0x7f030000

    invoke-virtual {p0, p1}, Lsg/vantagepoint/uncrackable1/MainActivity;->setContentView(I)V

    return-void
.end method
```

После изменения, нам надо заново собрать и подписать приложение. Собираем командой

`apktool b base`

Чтобы подписать, генерируем keystore

```
keytool -genkey -v -keystore my-release-key.keystore -alias alias_name -keyalg RSA -keysize 2048 -validity 10000
```

И собственно подписываем

```
jarsigner -verbose -sigalg SHA1withRSA -digestalg SHA1 -keystore my-release-key.keystore my_application.apk alias_name
```

Запускаем, и теперь нас больше не беспокоит никакое окно!

![](/assets/images/ru/owasp-1/3.png)


Дальше нас ждет input, вводим что-нибудь и видим еще одно окно

![](/assets/images/ru/owasp-1/4.png)

Чтобы найти код, отвечающий за проверку ввода, делаем поиск по исходникам и находим функцию, которая отвечает за вывод этого окна  - `.method public verify(Landroid/view/View;)V`, открываем и видим

```java
    invoke-static {p1}, Lsg/vantagepoint/uncrackable1/a;->a(Ljava/lang/String;)Z

    move-result p1

    if-eqz p1, :cond_0

    const-string p1, "Success!"

    invoke-virtual {v0, p1}, Landroid/app/AlertDialog;->setTitle(Ljava/lang/CharSequence;)V

    const-string p1, "This is the correct secret."

    :goto_0
    invoke-virtual {v0, p1}, Landroid/app/AlertDialog;->setMessage(Ljava/lang/CharSequence;)V

    goto :goto_1

    :cond_0
    const-string p1, "Nope..."

    invoke-virtual {v0, p1}, Landroid/app/AlertDialog;->setTitle(Ljava/lang/CharSequence;)V

    const-string p1, "That\'s not it. Try again."
```

Здесь, в зависимости от результат выполнения функции `Lsg/vantagepoint/uncrackable1/a;->a(Ljava/lang/String;)` Выводится Success либо Try again. Откроем эту функцию. В ней прменяется base64 к некой строки и далее она сравнивается. 

```java
.method public static a(Ljava/lang/String;)Z
    .locals 5

    const-string v0, "8d127684cbc37c17616d806cf50473cc"

    const-string v1, "5UJiFctbmgbDoLXmpL12mkno8HT4Lv8dlat8FxR2GOc="

    const/4 v2, 0x0

    invoke-static {v1, v2}, Landroid/util/Base64;->decode(Ljava/lang/String;I)[B

    move-result-object v1

    new-array v2, v2, [B

    :try_start_0
    invoke-static {v0}, Lsg/vantagepoint/uncrackable1/a;->b(Ljava/lang/String;)[B

    move-result-object v0

    invoke-static {v0, v1}, Lsg/vantagepoint/a/a;->a([B[B)[B

    move-result-object v0
    :try_end_0
    .catch Ljava/lang/Exception; {:try_start_0 .. :try_end_0} :catch_0

    goto :goto_0

    :catch_0
    move-exception v0

    const-string v1, "CodeCheck"

    new-instance v3, Ljava/lang/StringBuilder;

    invoke-direct {v3}, Ljava/lang/StringBuilder;-><init>()V

    const-string v4, "AES error:"

    invoke-virtual {v3, v4}, Ljava/lang/StringBuilder;->append(Ljava/lang/String;)Ljava/lang/StringBuilder;

    invoke-virtual {v0}, Ljava/lang/Exception;->getMessage()Ljava/lang/String;

    move-result-object v0

    invoke-virtual {v3, v0}, Ljava/lang/StringBuilder;->append(Ljava/lang/String;)Ljava/lang/StringBuilder;

    invoke-virtual {v3}, Ljava/lang/StringBuilder;->toString()Ljava/lang/String;

    move-result-object v0

    invoke-static {v1, v0}, Landroid/util/Log;->d(Ljava/lang/String;Ljava/lang/String;)I

    move-object v0, v2

    :goto_0
    new-instance v1, Ljava/lang/String;

    invoke-direct {v1, v0}, Ljava/lang/String;-><init>([B)V

    invoke-virtual {p0, v1}, Ljava/lang/String;->equals(Ljava/lang/Object;)Z

    move-result p0

    return p0
.end method
```

Давайте напишем патч, чтобы посмотреть, с какой строкой сравнивается наш ввод. Для этого создадим просто андроид приложение, с таким кодом, оно просто выводит окно, текстом

```java
public class MainActivity extends AppCompatActivity {

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        Toast.makeText(getApplicationContext(),"Hello Javatpoint",Toast.LENGTH_SHORT).show();
    }
}
```

Переведем это в smali код

```java
const-string v0, "Hello Javatpoint"

    const/4 v1, 0x0

    invoke-static {p1, v0, v1}, Landroid/widget/Toast;->makeText(Landroid/content/Context;Ljava/lang/CharSequence;I)Landroid/widget/Toast;

    move-result-object p1

    invoke-virtual {p1}, Landroid/widget/Toast;->show()V
```

Берем этот код, и вставляем в метод, где сравнивается секретная строка, с нашим вводом

```java

# direct methods
.method public static a(Ljava/lang/String;)Z
    .locals 5

    const-string v0, "8d127684cbc37c17616d806cf50473cc"

    const-string v1, "5UJiFctbmgbDoLXmpL12mkno8HT4Lv8dlat8FxR2GOc="

    const/4 v2, 0x0

    invoke-static {v1, v2}, Landroid/util/Base64;->decode(Ljava/lang/String;I)[B

    move-result-object v1

    new-array v2, v2, [B

    :try_start_0
    invoke-static {v0}, Lsg/vantagepoint/uncrackable1/a;->b(Ljava/lang/String;)[B

    move-result-object v0

    invoke-static {v0, v1}, Lsg/vantagepoint/a/a;->a([B[B)[B

    move-result-object v0
    :try_end_0
    .catch Ljava/lang/Exception; {:try_start_0 .. :try_end_0} :catch_0

    goto :goto_0

    :catch_0
    move-exception v0

    const-string v1, "CodeCheck"

    new-instance v3, Ljava/lang/StringBuilder;

    invoke-direct {v3}, Ljava/lang/StringBuilder;-><init>()V

    const-string v4, "AES error:"

    invoke-virtual {v3, v4}, Ljava/lang/StringBuilder;->append(Ljava/lang/String;)Ljava/lang/StringBuilder;

    invoke-virtual {v0}, Ljava/lang/Exception;->getMessage()Ljava/lang/String;

    move-result-object v0

    invoke-virtual {v3, v0}, Ljava/lang/StringBuilder;->append(Ljava/lang/String;)Ljava/lang/StringBuilder;

    invoke-virtual {v3}, Ljava/lang/StringBuilder;->toString()Ljava/lang/String;

    move-result-object v0
   
    const/4 v1, 0x0

    invoke-static {p1, v0, v1}, Landroid/widget/Toast;->makeText(Landroid/content/Context;Ljava/lang/CharSequence;I)Landroid/widget/Toast;

    move-result-object p1

    invoke-virtual {p1}, Landroid/widget/Toast;->show()V

    invoke-static {v1, v0}, Landroid/util/Log;->d(Ljava/lang/String;Ljava/lang/String;)I

    move-object v0, v2

    return p0
.end method
```

Мы пропатчили программу кодом, который выводит строку, которая является флагом к решению. Собераем наше приложение и запустим.

![](/assets/images/ru/owasp-1/10.png)

![](/assets/images/ru/owasp-1/11.png)

Вот и флаг! На этом расход