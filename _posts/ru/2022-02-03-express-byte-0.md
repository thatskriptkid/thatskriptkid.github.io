---
layout: post
title: ExpressБайтоебство_0
tags: [windows]
category: [ru]
---

*Быстрый рассказ о хламе, что скапливается в голове*

# 0

Увидел [твит](https://twitter.com/ShadowChasing1/status/1494470323063779353)

![](/assets/images/ru/express_0/twit.png)

Стало интересно, малварь для турков, на момент просмотра было всего 5 детектов на [вт](https://www.virustotal.com/gui/file/6ee1e629494d7b5138386d98bd718b010ee774fe4a4c9d0e069525408bb7b1f7/detection). Почему так мало? (когда сделал скрин, стало уже 10)

![](/assets/images/ru/express_0/vs.png)

В письме присылался ISO файл, внутри lnk и DLL. ISO обычно используется для избегания детектирования. lnk вызывает функцию DeleteDateConnectionPosition в DLL. Анализирую DLLы я так - 
1. Копирую в папку саму длл и файл rundll32.exe
2. В x64dbg в commandline прописываю ```"D:\rundll32.exe" D:\\malware.dll,DeleteDateConnectionPosition```
3. В preference ставлю галочку Dll Entry (чтобы словить нашу длл)
4. Кликаю F9, пока не подгрузится наша длл
5. Как подгрузилась, ставлю брейкпоинт на нужную функцию

на первый взгляд все выглядело очень мутно, куча неочевидных действий, но есть VirtualAlloc. VirtualAlloc обычно используется для распаковки в выделенную память чегото

![](/assets/images/ru/express_0/imports.png)

выявился одинаковый повторяющийся паттерн through бинарник, который очень похож на декодирование чегото

![](/assets/images/ru/express_0/decode_logic.png)

mov-хуюв, чтото кудато перекладывается, мне было неважно, меня привлек VirtualAlloc, стало понятно, что чтото декодируется, а потом исполняется поэтому решено было тупо промотать все до прямого вызова адреса в памяти (типа call MEMORY_ADDR). И яего нашел

![](/assets/images/ru/express_0/call_rax.png)

Декодированые инструкции выглядили вот так

![](/assets/images/ru/express_0/decoded_instructions.png)

Чуть продебажил и увидел в памяти что-то похожее на PE

![](/assets/images/ru/express_0/pe.png)

Хэдер у него правда MZARUH, бросающийся в глаза. Это слово попадается в [yara правиле](https://github.com/avast/ioc/blob/master/CobaltStrike/yara_rules/cs_rules.yar) от avast, под именем `cobaltstrike_beacon_encoded`. Остановившись в дебагере запустил [hollows_hunter](https://github.com/hasherezade/hollows_hunter), дампнул этот PE, посмотрел свойства, намеков на cobaltstrike стало больше, да и ребята подтвердили

![](/assets/images/ru/express_0/cs.png)

# 1

Увидел [твит](https://twitter.com/RedDrip7/status/1493905786354892801).

![](/assets/images/ru/express_0/twit2.png)

[2 детекта](https://www.virustotal.com/gui/file/a4afaa41383f447d96d0ebb1e2e50721af080e951d40754a836215fb2c3f0660/detection). MSI. Распаковываем его любым способом из интернета, получаем mtSilverclient.exe. Это C#. Коннектится к snapsvcvirtual[.]net (0 детектов вт). Очень подозрительно. 

Мимикрирует под silverlight прогу

![](/assets/images/ru/express_0/silver.png)

Код разбит по тематическим неймспейсам, например hibiscus.contention занимается связью с С2. HelpMeOut - отправка/прием данных, которые кодируются ксором

![](/assets/images/ru/express_0/helpme.png)

ProcessHelp класс может рассказать обо всем функционале малвари

![](/assets/images/ru/express_0/processHelp.png)

Выполнение комманд происходит так, создает пайпы для powershell, и тупо работает через него.

![](/assets/images/ru/express_0/pipes.png)

интересно, почему так мало детектов? Я думаю потому что
1. По сути делает 1 коннект к "безобидному" домену и ждет
2. Запакован в MSI

# 2 

![](/assets/images/ru/express_0/2_twit.png)

Строки лежат в чистом виде, коннектится к следующим адресам

![](/assets/images/ru/express_0/url.png)

и скачивает файл в 

![](/assets/images/ru/express_0/2_open_file.png)

Получается просто downloader

# 3

![](/assets/images/ru/express_0/turla/3.png)

https://twitter.com/BaoshengbinCumt/status/1495578841544531968

Турла. Детектируется как Kazuar, по следующим yara правилам ([1](https://github.com/Neo23x0/signature-base), [2](https://github.com/intezer/yara-rules)) - `APT_MAL_RU_Turla_Kazuar_May20_1`, `Kazuar`. Как известно, казуар - это .NET RAT.

сэндбоксы, windows loader, парсеры проанализровать файл не могут

![](/assets/images/ru/express_0/turla/4.png)

Раз так, откроем его в хекс редакторе, посмотрим глазами. Мы увидим, что один файл содержит внутри себя как минимум три PE (судя по хэдеру)

![](/assets/images/ru/express_0/turla/5.png)

Разделим файл руками на 3 части, каждая будет являться PE файлом. Первый, невалидный, его не может загрузить windows loader и парсеры ругаются.

![](/assets/images/ru/express_0/turla/7.png)

Дальше начал читать что пишет [чувак](https://www.red-gate.com/simple-talk/blogs/anatomy-of-a-net-assembly-pe-headers/) по поводу структуры .NET файлов, так как они очевидно отличаются от стандартной PE. Отличаются например тем, что у .NET файлов всего 3 секции и таблица импортов и тд лежит в секции text.text вообще содержит всю CLR метадату. 

![](/assets/images/ru/express_0/turla/8.png)

PEbear с недавних пор понимает .NET файлы. У нашего файла он пустой

![](/assets/images/ru/express_0/turla/9.png)

Короче я решил начать с того, чтобы восстановить CLI header (заголовок, описывающий .NET файл). Смотрите, секция .text валидного .NET файла должна начинаться по оффсету 0x200, и с 0x208 начинается CLI header. Я взял рандомный валидный .net файл, и сравнил с нашим казуартом

![](/assets/images/ru/express_0/turla/10.png)

стало очевидно, что здесь вставлен огромный блоб заполненый нулям, нахрена он нужен, я незнаю, так как он рушит всю структуру и тут не нужен. Зная, что каждый cli header должен начинаться с байт 0x48 (его размер) 0x2 (константа) 0x5 (константа), сделал поиск по этим байтам дальше по файлу. Байты были найдену по смещению 0x2008, то есть файл был испорчен (хрен пойми кем и как) блобом заполненым нулями размером 0x1e00 

![](/assets/images/ru/express_0/turla/11.png)

Вырезаем этот блоб из файла. Почему он именно длиной 0x1e00? Если мы посмотрим на raw и virtualaddress секции text, то его разница как раз будет равняться 1e00

![](/assets/images/ru/express_0/turla/12.png)

Теперь сравним валидный и нашего бедолагу в dnspy

![](/assets/images/ru/express_0/turla/13.png)

Как видно, у нас все еще не определяются потоки и даже подпись. Потоки находятся в Metadata. Metadata RVA прописан в .NET header. Metadata header должен начинаться с константы 0x424a5342. В нашем файле сейчас 0x369A0. Что мы делаем? Мы ищем в нашем файле константу 0x424a5342 и подгоняем RVA так, чтобы он начинался с этой константы. Он становится равным - OriginalMetadata.VA - 0x1e00

![](/assets/images/ru/express_0/turla/15.png)

У нас определились потоки и даже типы (правда коряво)

![](/assets/images/ru/express_0/turla/16.png)

Теперь посмотрим на таблицу импортов. 

![](/assets/images/ru/express_0/turla/17.png)


У .NET должен быть 1 импорт с 1 функцией, выставляем правильно соответствующие оффсеты и получаем

![](/assets/images/ru/express_0/turla/18.png)

entry point должен указывать на начало loader stub, который представлен байтами FF 25 00 20 40 00 или по другому jmp 0x402000. Это jump на CorExe библиотеки mscoree. Поэтому ищем эти байты у себя в файле, высчитываем RVA и заменяем.

![](/assets/images/ru/express_0/turla/19.png)

по крайней мере теперь он запускается. И падает, что естественно

![](/assets/images/ru/express_0/turla/20.png)

а естественно, потому что .net runtime по всей видимости не может найти код и метаданные кода (MethodDef, TypeDef, ...). Мне еще удалось восстановить .reloc и .rsrc секции, но ирония в том, что они для нормальной работы не нужны. Точнее, их повреждение не мешает работе. .reloc секция всегда содержит 1 запись, это собственно говоря релокация адреса входа в функцию CorExe. Ну а в ресурсах обычно лежат два entity - ID и манифест. Попытки дальнейшего восстановления дали понять, что вручную это практически невозможно и я забил.

Несмотря на поврежденную структуру мы все еще может судить о функционале по видимым строкам в бинарнике. Как минимум таким функционалом обладает данный образец:

1. Использование шифрования (симметричное и RSA)
2. WMI запросы
3. Работа с сетью (прокси, HTTPS)
4. Process Injection/DLL injection (скорее всего для инжекта DLL в Explorer.exe)
5. Работа с реестром
6. Работа  с файловой системой
7. работа с XML
   
Внутри, бинарник содержит две DLL x86/x64. Они содержат метод проверки, заинжекчены ли они в Explorer.exe (Инжект в Explorer.exe является одним из [известных](https://unit42.paloaltonetworks.com/unit42-kazuar-multiplatform-espionage-backdoor-api-access/) индикаторов Turla Kazuar .NET)

![](/assets/images/ru/express_0/turla/21.png)

Очень интересная деталь, под конец, я нашел, что бинарник содержит строку ```E73DCA6A8E53934C77A0BB669FDC383A20925866``` - SHA1 хэш [известного образца](https://www.virustotal.com/gui/file/ae0e99f574dcecf015d10d5209dd80c6b1b63101a65497a83c041d69a4182220/details) турлы казуар, который был загружен на ВТ	2020-05-15. То есть, этот образец использует старые и известные пейлоды

![](/assets/images/ru/express_0/turla/22.png)


Вывод - в целом, несмотря на то, что неудалось восстановить образец до работоспособного состояния, по сумме признаков можно сделать вывод о совпадении публично описанного функционала с имеющимся.


