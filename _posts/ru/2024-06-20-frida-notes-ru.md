---
layout: post
title: Заметка по Frida
tags: [note]
category: [ru]
---

# Frida note

Это не полноценная статья! Это мои заметки по изучению фриды. Когда нужно быстро вспомнить какой-то аспект фриды, можно будет к ним быстро вернуться. 

## Frida General

Frida - это Open Source Dynamic Code Instrumentation Framework. Instrumentation (инструментация) - это модификация программы для сбора данных о ее поведении, производительности или других характеристиках. Frida работает с процессом в рантайме, вмешиваясь в его работу. Возможности Frida:

1. API tracing
2. Логгировать действия уже запущенного приложения, не останавливая его работы
3. Расшифровка TLS трафика
4. Фаззинг функций изнутри процесса (в отличии от классических фазеров)
5. Дамп памяти процесса
6. Создание протектора, антидебаггера (!)
7. etc…

Фриду можно воспринимать как заскриптованный дебаггер. Она также как и дебагер подключается к процессу, но позволяет выполнять некоторые действия быстрее дебагера. Принцип работы Фриды заключается в инжекте в процесс QuickJS. QuickJS - маленький, самодостаточный интерпретатор Javascript. Вы пишите Frida скрипт на JS, а интерпретатор внутри процесса его исполняет. Если нет рута (ios, android), то фриду можно добавить непосредственно в исследуемое приложение следующими способами:

1. Модификация исходников
2. Патчинг бинарника 
3. Динамическая подгрузка (LD_PRELOAD)

## Frida mini examples

Папка `frida_mini_examples` содержит примеры возможностей фриды.

`hello.c`:

```c
#include <stdio.h>
#include <unistd.h>

void f (int n)
{
  printf ("Number: %d\n", n);
}

int main (int argc, char * argv[])
{
  int i = 0;

  printf ("f() is at %p\n", f);

  while (1)
  {
    f (i++);
    sleep (1);
  }
}
```

`hook.py 0xaddr` - перехватывает функцию `f()` и выводит аргумент

```javascript
import frida
import sys

# intercept func at addr from arg
session = frida.attach("hello")
script = session.create_script("""
Interceptor.attach(ptr("%s"), {
    onEnter(args) {
        send(args[0].toInt32()); // print func args
    }
});
""" % int(sys.argv[1], 16))
def on_message(message, data):
    print(message)
script.on('message', on_message)
script.load()
sys.stdin.read()
```

`modify.py 0xaddr` - модифицирует первый аргумент, меняет его на 1337

```javascript
import frida
import sys

session = frida.attach("hello")
script = session.create_script("""
Interceptor.attach(ptr("%s"), {
    onEnter(args) {
        args[0] = ptr("1337");
    }
});
""" % int(sys.argv[1], 16))
script.load()
sys.stdin.read()
```

`call_funcs.py 0xaddr` - вызов внутренней функции f()

```javascript
import frida
import sys

session = frida.attach("hello")
script = session.create_script("""
const f = new NativeFunction(ptr("%s"), 'void', ['int']);
f(1911);
f(1911);
f(1911);
""" % int(sys.argv[1], 16))
script.load()
```

`inject_string` - создает строку "TESTMEPLZ!" в адресном пространстве процесса hi и вызывает функцию f() с этой строкой в качестве аргумента

`hi.c`:

```c
#include <stdio.h>
#include <unistd.h>

int f (const char * s)
{
  printf ("String: %s\n", s);
  return 0;
}

int main (int argc, char * argv[])
{
  const char * s = "Testing!";

  printf ("f() is at %p\n", f);
  printf ("s is at %p\n", s);

  while (1)
  {
    f (s);
    sleep (1);
  }
}
```

```javascript
import frida
import sys

session = frida.attach("hi")
script = session.create_script("""
const st = Memory.allocUtf8String("TESTMEPLZ!");
const f = new NativeFunction(ptr("%s"), 'int', ['pointer']);
f(st);
""" % int(sys.argv[1], 16))
def on_message(message, data):
    print(message)
script.on('message', on_message)
script.load()
```

`inject_struct` - client коннектится к 127.0.0.1 по порту 5000, после нажатия Enter. Скрипт подменяет структуру sockaddr на собственную

`nc -lp 5000`
`nc -lp 5001`
`./client 127.0.0.1`

`client.c`

```c
#include <arpa/inet.h>
#include <errno.h>
#include <netdb.h>
#include <netinet/in.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <sys/socket.h>
#include <sys/types.h>
#include <unistd.h>

int main (int argc, char * argv[])
{
  int sock_fd, i, n;
  struct sockaddr_in serv_addr;
  unsigned char * b;
  const char * message;
  char recv_buf[1024];

  if (argc != 2)
  {
    fprintf (stderr, "Usage: %s <ip of server>\n", argv[0]);
    return 1;
  }

  printf ("connect() is at: %p\n", connect);

  if ((sock_fd = socket (AF_INET, SOCK_STREAM, 0)) < 0)
  {
    perror ("Unable to create socket");
    return 1;
  }

  bzero (&serv_addr, sizeof (serv_addr));

  serv_addr.sin_family = AF_INET;
  serv_addr.sin_port = htons (5000);

  if (inet_pton (AF_INET, argv[1], &serv_addr.sin_addr) <= 0)
  {
    fprintf (stderr, "Unable to parse IP address\n");
    return 1;
  }
  printf ("\nHere's the serv_addr buffer:\n");
  b = (unsigned char *) &serv_addr;
  for (i = 0; i != sizeof (serv_addr); i++)
    printf ("%s%02x", (i != 0) ? " " : "", b[i]);

  printf ("\n\nPress ENTER key to Continue\n");
  while (getchar () == EOF && ferror (stdin) && errno == EINTR)
    ;

  if (connect (sock_fd, (struct sockaddr *) &serv_addr, sizeof (serv_addr)) < 0)
  {
    perror ("Unable to connect");
    return 1;
  }

  message = "Hello there!";
  if (send (sock_fd, message, strlen (message), 0) < 0)
  {
    perror ("Unable to send");
    return 1;
  }

  while (1)
  {
    n = recv (sock_fd, recv_buf, sizeof (recv_buf) - 1, 0);
    if (n == -1 && errno == EINTR)
      continue;
    else if (n <= 0)
      break;
    recv_buf[n] = 0;

    fputs (recv_buf, stdout);
  }

  if (n < 0)
  {
    perror ("Unable to read");
  }

  return 0;
}
```

`inject_struct.py`

```javascript
import frida
import sys

session = frida.attach("client")
script = session.create_script("""
send('Allocating memory and writing bytes...');
const st = Memory.alloc(16);
st.writeByteArray([0x02, 0x00, 0x13, 0x89, 0x7F, 0x00, 0x00, 0x01, 0x30, 0x30, 0x30, 0x30, 0x30, 0x30, 0x30, 0x30]);
// 0x1389 = 5001d
Interceptor.attach(Module.getExportByName(null, 'connect'), {
    onEnter(args) {
        send('Injecting malicious byte array:');
        args[1] = st;
    }
    //, onLeave(retval) {
    //   retval.replace(0); // Use this to manipulate the return value
    //}
});
""")

def on_message(message, data):
    if message['type'] == 'error':
        print("[!] " + message['stack'])
    elif message['type'] == 'send':
        print("[i] " + message['payload'])
    else:
        print(message)
script.on('message', on_message)
script.load()
sys.stdin.read()
```

## Frida windows

### Пример 1

В папке `samples_for_demonstration` лежат образцы классического plugx.

Запускать:

`frida -l winapi_hooks.py -f sample.exe`

`winapi_hooks.py` - Скрипт хукает функции VirtualProtect, VirtualAlloc, CreateFileW, CreateFileA и выводит их аргументы. Делает дамп памяти, к которой применяется VirtualProtect.

```javascript
var vp = Module.getExportByName(null, "VirtualProtect");


Interceptor.attach(vp, {
    onEnter: function(args)
    {
		var vpAddress = args[0];
		var vpAddr = args[0];
		var vpSize = args[1].toInt32();
        console.log("VirtualProtect called.\n Address:" + vpAddr + " Size:" + vpSize);
	
		console.log(hexdump(vpAddress));
		
		var dump = vpAddress.readByteArray(vpSize);
		var filename = "D:\\dumps\\" + vpAddress + "_dump.bin";
		var file = new File(filename, "wb");
		file.write(dump);
		file.flush();
		file.close();
		
    }
});


var va = Module.getExportByName(null, "VirtualAlloc");

Interceptor.attach(va, {
    onEnter: function(args)
    {
		
		var vaSize = args[1].toInt32();
		var vaProtect = args[3];
        console.log("VA called.\n Size:" + vaSize + " Protect:" + vaProtect);
		
    },
	onLeave: function(retVal)
	{
		console.log("VA ret:" + retVal);
	}
	
});

var createFileWAddr = Module.getExportByName(null, "CreateFileW");
 Interceptor.attach(createFileWAddr, {
        onEnter: function(args) {
            console.log('CreateFileW() call');
            console.log('path:' + args[0].readUtf16String());
            }
    });

var createFileAAddr = Module.getExportByName(null, "CreateFileA");
 Interceptor.attach(createFileAAddr, {
        onEnter: function(args) {
            console.log('CreateFileA() call');
            console.log('path:' + args[0].readAnsiString());
            }
    });	

```

![](/assets/images/ru/frida_notes/plugx_1.png)

![](/assets/images/ru/frida_notes/plugx_2.png)

### Пример 2

Я скачал рандомный образец (RemcosRAT) с [malware bazaar](https://bazaar.abuse.ch/sample/0e43af051e536e7c731f3d856baaf644e474c4c80ea22cb0fb4386d42c2e1056/):

![](/assets/images/ru/frida_notes/win_1.png)

Попробуем посмотреть, что делает малварь с файлами следующим скриптом фрида:

```js
var createFileWAddr = Module.getExportByName(null, "CreateFileW");
 Interceptor.attach(createFileWAddr, {
        onEnter: function(args) {
            console.log('path:' + args[0].readUtf16String());
            }
    });
```

В результате получим открываемые файлы малварью:

![](/assets/images/ru/frida_notes/win_2.png)

Среди путей видим интересный файл `install.vbs`, который лежит в папке `%TEMP%`. Откроем эту папку и видим, что созданный файл отсутствует.  

![](/assets/images/ru/frida_notes/win_3.png)

Можно предположить, что он после использования удаляется. Как нам поступить? Мы можем перехватить записываемые данные в файл и сохранить их до удаления. Для этого напишем скрипт перехвата вызова `WriteFile`, будем сохранять буфер в отдельный файл:

```js
var writeFileAddr = Module.getExportByName(null, "WriteFile");
 Interceptor.attach(writeFileAddr, {
        onEnter: function(args) {
			
			var buff = args[1];
			var size = args[2].toInt32();
			console.log("WriteFile | size = " + size)
			
			console.log(hexdump(buff));
			
			var dump = buff.readByteArray(size);
			var filename = "D:\\dumps\\" + Math.random() + "_dump.bin";
			var file = new File(filename, "wb");
			file.write(dump);
			file.flush();
			file.close();
            }
    });
```
В результате мы сохранили целевой скрипт, который и вправду самоудаляется после выполнения:

![](/assets/images/ru/frida_notes/win_4.png)


## Дамп памяти Android приложения

С помощью Frida можно дампнуть память любого процесса. 

1. Скачиваем любую apk, в нашем случае это будет mcdonalds
   
2. Устанавливаем apk 
   
   `adb install mcodnalds.apk`

3. Копируем фрида сервер на эмулятор/телефон и запускаем его

`adb push frida-server-12.7.26-android-x86_64 /data/local/tmp/frida`

`chmod +x /data/local/tmp/frida`

`/data/local/tmp/frida &`

4. Устанавливаем fridump
   
   `git clone https://github.com/Nightbringer21/fridump.git` 

5. Открываем приложение mcdonalds. Вводим в поле пароль/телефон и нажимаем Войти!

![](/assets/images/ru/frida_notes/mcdonalds_1.jpg)

6. Делаем дамп процесса. Дампы сохранятся в папку `dump`

    `python fridump.py -U -v -s mcdonalds`

7. Ищем в дампе наш пароль 

    `grep -arni "pass123" dump/*`

![](/assets/images/ru/frida_notes/mcdonalds_2.jpg)


## Frida Android example aka UnCrackable App for Android Level 2

Описание задания второго уровня

<pre>
This app holds a secret inside. May include traces of native code.

Objective: A secret string is hidden somewhere in this app. Find a way to extract it.
</pre>

Описание уже содержит хорошую подсказку, но не будем забегать вперед. Скачиваем apk, устанавливаем на android эмулятор, запускаем. Видим, как и в первом задании, детектирование рута

![](/assets/images/ru/frida_notes/1.png)

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

![](/assets/images/ru/frida_notes/2.png)

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

С помощью Frida, мы можем заставить эти функции всегда возвращать false и обходить проверку. Для этого нам необходимо написать скрипт на [javascript](/assets/images/ru/frida_notes/4.jpg). Чтобы работать с java классами, мы должны использовать обертку, для нашего кода (подробнее о API к java, читайте в [официальной документации](https://frida.re/docs/javascript-api/#java))

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

![](/assets/images/ru/frida_notes/3.png)

Мы получили доступ к вводу секретной строки. Вводим что-нибудь

![](/assets/images/ru/frida_notes/5.png)

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

Этот код говорит нам о том, что приложение загружает библиотеку с именем _foo_. Если мы распакуем наше приложение, то в папке _lib_ мы видим библиотеку _libfoo.so_. Теперь пришло время изучить ее. Так как у нас нет исходных кодов библиотеки и написана она на С/С++, то нам необходимо воспользоваться дизассемблером, в нашем случае - [Binary Ninja](https://binary.ninja/). Можно использовать бесплатную [Веб версию](https://cloud.binary.ninja/).

Открываем файл нашей библиотеки и находим в списке нашу функцию `bar`

![](/assets/images/ru/frida_notes/6.png)

Открываем функцию. Углубляться в анализ ассемблерного кода мы сегодня не будем, возможно в другой жизни это сделаем. Просто обратите внимание на выделенный код и в особенности на hex значения

![](/assets/images/ru/frida_notes/7.png)

на адреса они не похожи, а похожи они на строки. Попробуем посмотреть опцией "отобразить, как символы"

![](/assets/images/ru/frida_notes/8.png)

Получилось вот так и это похоже на правду

![](/assets/images/ru/frida_notes/9.png)

Проделаем для всех значений

![](/assets/images/ru/frida_notes/10.png)

Эти части складываются в единную строку "Thanks for all the fish". Далее по коду мы видим сравнение нашего ввода с этой строкой

![](/assets/images/ru/frida_notes/11.png)

Проверим строку в приложении

![](/assets/images/ru/frida_notes/12й.png)

Это и есть наша секретная строка!

# Frida TLS

[Подробное обьяснение принципа работы](https://b.poc.fun/decrypting-schannel-tls-part-1/)

С помощью Frida можно расшифровывать schannel TLS трафик (IIS, RDP, IE, Outlook, Powershell, LDAP ...). Для этого нам понадобится скрипт [keylog.js](https://github.com/ngo/win-frida-scripts/blob/master/lsasslkeylog-easy/keylog.js), wireshark и для демонстрации будем использовать Windows 11. 

SChannel a.k.a Secure Channel - это подсистема windows, используемая приложениями при работе с TLS (установка соединения, прием соединения от клиента).

keylog файл содержит значения, которые используются для генерации сессионых ключей TLS. Обьяснение данных в логе можно прочитать [тут](https://www.ietf.org/archive/id/draft-thomson-tls-keylogfile-00.html)

![](/assets/images/ru/frida_notes/tls_7.png)

Скрипт хукает функции библиотеки `ncrypt.dll`. Например, функция `SslHashHandshake` используется для генерации хэша во время SSL Handshake.

```javascript
var shh = Module.findExportByName('ncrypt.dll', 'SslHashHandshake');
if(shh != null){
	Interceptor.attach(shh, {
```

Дампает входящие аргументы функций связанных с TLS в кейлог файл, который затем добавляется в Wireshark.

1. Чтобы фрида смогла приатачится к lsass.exe, в настройках `Exploit protection` выставляем `hardware-enforced stack protection off`.

![](/assets/images/ru/frida_notes/tls_3.png)
 
2. В скрипте `keylog.js` указываем путь к логу. Узнаем PID процесса lsass.exe и аттачимся к нему командой

`frida -p PID -l SCRIPT`

![](/assets/images/ru/frida_notes/tls_2.png)

3. Открываем Wireshark 

4. Делаем какой либо HTTS запрос. Например:

`Invoke-WebRequest -Uri "https://www.google.de" -Method GET -TimeoutSec 5`

Убеждаемся, что видим только зашифрованный трафик.

5. Файл лога `keylog.log` должен заполниться значениями. 
   
6. В Wireshark, в настройках `Edit->Preferences->Protocols->TLS` вводим путь к логу в `(Pre)-Master-Secret log filename`

7. Трафик, который был у нас в Wireshark расшифруется.

![](/assets/images/ru/frida_notes/tls_1.png)

Чистый запрос Invoke-WebRequest

![](/assets/images/ru/frida_notes/tls_4.png)

Чистый ответ гугла

![](/assets/images/ru/frida_notes/tls_5.png)

# Полезные ссылки

Для [2600 митапа](https://t.me/ast2600/333) я подготавливал [презентацию](https://github.com/thatskriptkid/thatskriptkid.github.io/blob/master/assets/files/2600/%D0%A7%D1%82%D0%BE%20%D1%82%D0%B0%D0%BA%D0%BE%D0%B5%20Frida_%202600.pdf) по возможностям фриды. 

[Android greybox fuzzing with AFL++ Frida mode](https://blog.quarkslab.com/android-greybox-fuzzing-with-afl-frida-mode.html)