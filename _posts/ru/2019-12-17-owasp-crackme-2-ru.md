---
layout: post
title: Решаем "OWASP UnCrackable App for Android Level 2", с помощью FRIDA и Binary Ninja!
tags: [android, crackme]
category: [ru]
---

## Intro

[Первая часть тут]({% post_url ru/2019-12-12-owasp-crackme-1-ru %}). Советую начать с нее, так как многие моменты описаные там, здесь описываться не будут.

Tools used: [FRIDA](https://frida.re/), [Binary Ninja](https://binary.ninja/)

Продолжаем решать задания [от OWASP](https://github.com/OWASP/owasp-mstg/tree/master/Crackmes). Эти задания даются в качестве демонстративных примеров в [OWASP Mobile Security Testing Guide](https://www.owasp.org/index.php/OWASP_Mobile_Security_Testing_Guide). Я не читал этот гайд и буду пытаться сделать задания, опираясь только на свои обрывки знаний об андроиде. Скорей всего есть более быстрые и короткие пути решения. Решение ниже просто является моим.

## UnCrackable App for Android Level 2

Описание задания второго уровня

<pre>
This app holds a secret inside. May include traces of native code.

Objective: A secret string is hidden somewhere in this app. Find a way to extract it.
</pre>

Описание уже содержит хорошую подсказку, но не будем забегать вперед. Скачиваем apk, устанавливаем на android эмулятор, запускаем. Видим, как и в первом задании, детектирование рута

![](/assets/images/ru/owasp-2/1.png)

Давайте обойдем эти проверки другим способом (не выпиливанием кода из apk), с помощью [Frida](https://frida.re/). Это фреймворк, который позволяет вмешиваться в работу приложения, во время выполнения. Инструкция по установке проста:

```
pip install frida-tools
```

Далее нам необходимо скачать подходящий сервер frida [отсюда](https://github.com/frida/frida/releases). У меня эмулятор android x86, поэтому я скачал архив с названием _frida-server-12.7.26-android-x86.xz_. Распаковываем и закидываем на эмулятор

```
adb push frida-server-12.7.26-android-x86_64 /data/local/tmp/frida
```

Запускаем

```
chmod +x /data/local/tmp/frida
/data/local/tmp/frida &
```

После этого, мы должны командой `frida-ps -U` получить список процессов на эмуляторе, чтобы убедиться, что Frida установлена правильно и сервер запущен

![](/assets/images/ru/owasp-2/2.png)

Теперь возвращаемся к приложению и детектированию рута. Оно по прежнему происходит при помощи трех функций, которые мы видим после `invoke-static`

```java
    invoke-static {}, Lsg/vantagepoint/a/b;->a()Z

    move-result v0

    if-nez v0, :cond_0

    invoke-static {}, Lsg/vantagepoint/a/b;->b()Z

    move-result v0

    if-nez v0, :cond_0

    invoke-static {}, Lsg/vantagepoint/a/b;->c()Z

    move-result v0
```

С помощью Frida, мы можем заставить эти функции всегда возвращать false и обходить проверку. Для этого нам необходимо написать скрипт на [javascript](/assets/images/ru/owasp-2/4.jpg). Чтобы работать с java классами, мы должны использовать обертку, для нашего кода (подробнее о API к java, читайте в [официальной документации](https://frida.re/docs/javascript-api/#java))

```javascript
Java.perform(function() {
    //our code...
})
```

Чтобы получить доступ к классу `sg.vantagepoint.a.b` (в котором находятся наши проверки), мы должны использовать метод `Java.use(classname)`

```javascript
Java.perform(function() {

	var antiRootClass = Java.use("sg.vantagepoint.a.b");
})
```

Теперь имея доступ к нужному классу, зная имя метода, мы можем переопределить его поведение на возврат false

```javascript
antiRootClass.a.implementation = function() {
		return false;
	}
```

Повторим данный код, для всех трех методов и в итоге получим 

```javascript
console.log("Script started...");

Java.perform(function() {

	var antiRootClass = Java.use("sg.vantagepoint.a.b");
	antiRootClass.a.implementation = function() {
		return false;
	}
	console.log("sg.vantagepoint.a.b.a modified");
	
	antiRootClass.b.implementation = function() {
		return false;
	}
	console.log("sg.vantagepoint.a.b.b modified");
	
	antiRootClass.c.implementation = function() {
		return false;
	}
	console.log("sg.vantagepoint.a.b.c modified");
})

```

_Чтобы запустить этот скрипт, вам необходимо знать package name приложения. Его можно получить из манифеста или запустив приложение и посмотрев все той же командой `frida-ps`._

Сохраняем код скрипта в файл с именем `owasp2.js` и выполняем такую команду 

`frida -U -l owasp2.js --no-paus -f owasp.mstg.uncrackable2`

которая сама стартует приложение и выполняет скрипт. В итоге мы не получаем окно, так как все наши проверки рута были отключены

![](/assets/images/ru/owasp-2/3.png)

Мы получили доступ к вводу секретной строки. Вводим что-нибудь

![](/assets/images/ru/owasp-2/5.png)

появилось окно. Ищем в исходниках место, где встречается эта строка и идет проверка. Таким местом оказался метод `verify()` в классе `MainActivity`. Его сокращенная версия

```java

    ...

    invoke-virtual {p1}, Landroid/widget/EditText;->getText()Landroid/text/Editable;

    move-result-object p1

    iget-object v1, p0, Lsg/vantagepoint/uncrackable2/MainActivity;->m:Lsg/vantagepoint/uncrackable2/CodeCheck;

    invoke-virtual {v1, p1}, Lsg/vantagepoint/uncrackable2/CodeCheck;->a(Ljava/lang/String;)Z

    move-result p1

    if-eqz p1, :cond_0

    const-string p1, "Success!"
```

Здесь мы видим, что введенный нами текст в `EditText`, отправляется в функцию `sg/vantagepoint/uncrackable2/CodeCheck;->a` и если все удачно, нам говорят "Success!". Очевидно, что именно там идет проверка нашего ввода на правильность. Откроем класс `sg.vantagepoint.uncrackable2.CodeCheck`

```java
.method private native bar([B)Z
.end method


# virtual methods
.method public a(Ljava/lang/String;)Z
    .locals 0

    invoke-virtual {p1}, Ljava/lang/String;->getBytes()[B

    move-result-object p1

    invoke-direct {p0, p1}, Lsg/vantagepoint/uncrackable2/CodeCheck;->bar([B)Z

    move-result p1

    return p1
.end method
```

Наш ввод, передается в функцию `bar()`

```java
invoke-direct {p0, p1}, Lsg/vantagepoint/uncrackable2/CodeCheck;->bar([B)Z
```

Обратите внимание на модификатор native функции

```java
.method private native bar([B)Z
```

Модификатор native означает, что реализация метода находится в библиотеках, написанных на других языках. Чтобы работать с такими библиотеками, используется механизм [JNI](https://developer.android.com/ndk/guides) (Java Native Interface). С помощью этого интерфейса, андроид приложение может вызывать код на С/С++. Значит, наше приложение использует внешнюю библиотеку. Найдем доказательство этому и сделаем поиск по исходникам, по слову _loadLibrary_. Находим следующий код в `MainActivity`

```java
# direct methods
.method static constructor <clinit>()V
    .locals 1

    const-string v0, "foo"

    invoke-static {v0}, Ljava/lang/System;->loadLibrary(Ljava/lang/String;)V

    return-void
.end method
```

Этот код говорит нам о том, что приложение загружает библиотеку с именем _foo_. Если мы распакуем наше приложение, то в папке _lib_ мы видим библиотеку _libfoo.so_. Теперь пришло время изучить ее. Так как у нас нет исходных кодов библиотеки и написана она на С/С++, то нам необходимо воспользоваться дизассемблером, в нашем случае - [Binary Ninja](https://binary.ninja/)! 

_*он платный, но у него также есть бесплатная web версия. вы можете использовать любой другой дизассемблер_

Открываем файл нашей библиотеки и находим в списке нашу функцию `bar`

![](/assets/images/ru/owasp-2/6.png)

Открываем функцию. Углубляться в анализ ассемблерного кода мы сегодня не будем, возможно в другой жизни это сделаем. Просто обратите внимание на выделенный код и в особенности на hex значения

![](/assets/images/ru/owasp-2/7.png)

на адреса они не похожи, а похожи они на строки. Попробуем посмотреть опцией "отобразить, как символы"

![](/assets/images/ru/owasp-2/8.png)

Получилось вот так и это похоже на правду

![](/assets/images/ru/owasp-2/9.png)

Проделаем для всех значений

![](/assets/images/ru/owasp-2/10.png)

Эти части складываются в единную строку "Thanks for all the fish". Далее по коду мы видим сравнение нашего ввода с этой строкой

![](/assets/images/ru/owasp-2/11.png)

Проверим строку в приложении

![](/assets/images/ru/owasp-2/12й.png)

Это и есть наша секретная строка!
