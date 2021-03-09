---
layout: post
title: Анализ шифровальщика XCRY написанного на Nim 
tags: [malware analysis]
category: [ru]
---

Не так давно, публично [стало известно](https://twitter.com/malwrhunterteam/status/1085962856728797184) о 32-битном шифровальщике написанном на языке Nim, под названием XCRY. Спасибо [MalwareHunterTeam!](https://twitter.com/malwrhunterteam)

[VirusTotal](https://www.virustotal.com/gui/file/e32c8b2da15e294e2ad8e1df5c0b655805d9c820e85a33e6a724b65c07d1a043/detection)

SHA-256: e32c8b2da15e294e2ad8e1df5c0b655805d9c820e85a33e6a724b65c07d1a043

[Скачать образец](/assets/files/e32c8b2da15e294e2ad8e1df5c0b655805d9c820e85a33e6a724b65c07d1a043.zip)

Образец не упакован и содержит много Debug информации. Имя шифровальщику было дано по тэгу добавляемому к зашифрованным файлам.

![](/assets/images/xcry/image19.png)

В сети есть [краткое описание](https://id-ransomware.blogspot.com/2019/01/xcry-ransomware.html) шифровальщика (некоторые моменты оттуда здесь описаны не будут), но не показана внутренняя работа. Цель данной статьи заполнить этот пробел. Как уже было сказано ранее, из-за присутствия в образце информации, оставленной после компиляции, мы можем видеть понятные имена функций. Вообще, данный факт позволяет нам понять как выглядит Main функция программ на Nim и в дальнейшем, если мы рассматриваем малварь скомпилированную без Debug информации, мы сможем выявить нужные функции. Для примера, на скрине слева - Xcry, справа рандомная малварь, тоже написанная на Nim, но скомпилированная без Debug. На нем визуально выделяются одинаковые функции

![](/assets/images/xcry/image24.png)

Кстати, чтобы такого не происходило, можете найти инструкцию по минимальной компиляции Nim программ в [этой статье](https://hookrace.net/blog/nim-binary-size/).

В функции PreMainInner() происходит инициализация всех используемых библиотек

![](/assets/images/xcry/image30.png)

Среди которых можно заметить библиотеку [nim libsodium](https://github.com/FedericoCeratto/nim-libsodium) - обертка над [libsoidum](https://libsodium.gitbook.io/doc/), которая является основой для работы шифровальщика.

![](/assets/images/xcry/image35.png)

Функция PreMainInner() очень важна, так как именно в ней мы можем посмотреть реальные импорты. 

Строки в Nim имеют свой кастомный тип - `NimString`. Они имеют динамический размер, поле с указанием длины строки и завершаются нулем. Также непосредственно перед строкой стоит байт 0x40

![](/assets/images/xcry/image18.png)

Строки в языке Nim могут быть созданы как объекты функцией [newObjRC1](https://github.com/nim-lang/Nim/blob/75dc69417a235c3be1edb3d89ddb49104bdd438d/lib/system/gc.nim#L453)

![](/assets/images/xcry/image25.png)

и как просто [память в куче](https://github.com/nim-lang/Nim/blob/662c5080755eb5a42df2c3acb84c044876571a46/lib/system/mm/boehm.nim#L13)

```nim
proc boehmAllocAtomic(size: int): pointer {.

  importc: "GC_malloc_atomic", boehmGC.}
```

Nim программы получают адреса используемых функций в рантайме. `nimLoadLibrary` - обертка над функцией `LoadLibrary`.

`add ecx, 8` - взятие “реальной” строки kernel32 (картинка выше) по смещению 8.

![](/assets/images/xcry/image4.png)

Для прямого вызова WinApi функций, в образце используется [библиотека Winim](https://github.com/khchen/winim), она облегчает написание кода. [Здесь](https://github.com/byt3bl33d3r/OffensiveNim) можно посмотреть примеры использования. Ниже можно увидеть получение адресов функций.

![](/assets/images/xcry/image6.png)


Весь этот код генерируется компилятором. Из-за этого статический анализ (просмотр импортов) дает неполную информацию. Возможно данный факт немного помогает в написании малвари на Nim. Например, `CreateProcessW` отсутствует в импортах, но используется во время работы во вредоносных целях, как и `FindFirstFile` для прохода по файлам.

Проверка повторного запуска происходит путем проверки существования файла `lock_file`. Данный файл создается по окончанию всей работы.

![](/assets/images/xcry/image17.png)
![](/assets/images/xcry/image23.png)

Xcry содержит публичный ключ, который используется библиотекой `libsodium`. Он хранится в памяти в виде строки и потом превращается в байтовый массив функцией `hex2bin` в `libsodium`

![](/assets/images/xcry/image31.png)

При старте копирует себя в `APPDATA` под рандомным именем.

![](/assets/images/xcry/image21.png)

Это имя генерируется функцией [randombytes](https://github.com/jedisct1/libsodium/blob/ae4add868124a32d4e54da10f9cd99240aecc0aa/src/libsodium/randombytes/randombytes.c%23L211) библиотеки `libsodium`

![](/assets/images/xcry/image9.png)

Интересно, что шифровальщик не перебирает подключенные диски, а имеет заранее приготовленную строку с потенциальными буквами диска. Начиная с буквы `Z`, которая должна соответствовать [сетевым дискам](https://en.wikipedia.org/wiki/Drive_letter_assignment).

![](/assets/images/xcry/image27.png)

Каждый найденный диск шифруется в отдельном потоке, который был создан функцией [Spawn](https://github.com/nim-lang/Nim/blob/56461c280f78c55f538da7f382e1c2c308e04915/doc/spawn.txt) в Nim.

Теперь переходим к основному функционалу. Для шифрования файлов используется алгоритм [XSalsa20](https://libsodium.gitbook.io/doc/advanced/stream_ciphers/xsalsa20) (отличается от Salsa20 более длинным nonce - 192 бита вместо 64). В начале, генерируется приватный ключ функцией [crypto_stream_keygen](https://libsodium.gitbook.io/doc/advanced/stream_ciphers/xsalsa20)

![](/assets/images/xcry/image1.png)

Этот ключ шифруется публичным ключом, функцией [crypto_box_seal](https://libsodium.gitbook.io/doc/public-key_cryptography/sealed_boxes).

![](/assets/images/xcry/image33.png)

В результате получается значение длиной `0xA1`

![](/assets/images/xcry/image11.png)

Далее это значение используется для генерации хэша (ShortHash), функцией [crypto_shorthash](https://libsodium.gitbook.io/doc/hashing/short-input_hashing) - которая возвращает случайные 16 байт

![](/assets/images/xcry/image10.png)

Затем, создается строка вида:

`Short Hash + пробел + private key + байт 0x0A`

![](/assets/images/xcry/image12.png)

Значение сохраняется в папку APPDATA под именем `encryption_key`.

![](/assets/images/xcry/image8.png)

Именно этот файл просят злоумышленники от жертвы, чтобы расшифровать данные за выкуп. То есть, у шифровальщика отсутствует функционал для работы с сетью и работа по отправке ключа на почту авторов лежит на жертве.

Что интересно, приватный ключ генерируется для каждого 1000 (0x3E8) файла (на скрине ниже - проверка модуля числа) и каждый такой ключ записывается в файл `encryption_key`.

![](/assets/images/xcry/image37.png)

Ниже показана структура файла `encryption_key` в виде `Kaitai Struct`

```yaml
meta:
  id: xcry_encryption_key
  file-extension: xcry_encryption_key
seq:
 - id: enc_key
   type: enc_key_struct
   repeat: eos
types:
  enc_key_struct:
    seq:
     - id: short_hash_key
       size: 16
     - id: inner_delimeter
       size: 1
     - id: private_key
       size: 0xa0
     - id: delimeter
       size: 1
```

![](/assets/images/xcry/image15.png)

Далее происходит чтение файла кусками по `0x2710` (10000) байт и его шифрование

![](/assets/images/xcry/image5.png)

Основная функция для шифрования - [crypto_secretbox_detached](https://libsodium.gitbook.io/doc/secret-key_cryptography/secretbox), которая использует сгенерированный ранее случайный ключ.

![](/assets/images/xcry/image36.png)

В начало каждого зашифрованного файла вставляет строка с `ShortHash`. Делается это для того, чтобы позже найти нужный приватный ключ для каждого конкретного файла, для расшифровки. На скрине - содержимое зашифрованного файла со строкой ShortHash

![](/assets/images/xcry/image32.png)

Тот же самый `ShortHash` в файле `encryption_key`

![](/assets/images/xcry/image16.png)

Оригинальный файл просто удаляется без перезаписывания, что позволяет просто восстановить файлы. Например вот способ затирания файлов, перед их удалением в open source шифровальщике [GonnaCry](https://github.com/tarcisio-marinho/GonnaCry/blob/a11630c94058dd1c2c91923b201be12f109541ec/src/GonnaCry/utils.py#L9) 

![](/assets/images/xcry/image3.png)

Также не удаляются теневые копии.

Вот и все! В итоге, мне было просто интересно посмотреть как выглядят внутренности программ на Nim и то, как может быть написан работающий шифровальщик на этом языке. Надеюсь, я понятно показал оба момента и спасибо, если дочитали до конца.  

Если вы хотите посмотреть на то как выглядят другие шифровальщики на Nim, можете обратить внимание на [атаку на форум IObit](https://www.bleepingcomputer.com/news/security/iobit-forums-hacked-to-spread-ransomware-to-its-members/) и соответствующий [образец](/assets/files/b53f222ffcc99939a1141a06e2240525c7154fcf2f39f8c5ca19a079e08a41fd.zip).