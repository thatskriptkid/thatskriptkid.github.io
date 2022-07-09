---
layout: post
title: Вычисление примерного (минимального) timestamp у golang elf malware
tags: [linux, malware analysis]
category: [ru]
---

## Вступление

Timestamp (или дата компиляции) является одним из главных характеристик малвари. Он позволяет понять является ли образец свежим, использовался ли образец в других атаках и дать представление о том, когда примерно готовилась атака. Но что делать, если малварь формата elf, который как известно не сохраняет дату компиляции файла?

Golang малварь получила большое распространение за последние несколько лет. 
<p>
<details>
<summary> список </summary>
<pre>
1. Sandworm-Team (Exaramel-Linux)
2. APT28 Zebrocy
3. Ekans
4. Kinsing (miner)
5. Mustang Panda 
6. Rocke
7. APT29 WellMess/WellMail
8. Gobot
9. rocke
10. htran
11. goodor
12. FritzFrog
13. Gobrut
14. Geacon
15. Kaiji
16. NSPPS
17. Carbanak
18. Veil
19. AgeLocker
20. Notrobin
21. r2r2
22. Blackrota
23. IPStorm
24. RobbinHood
25. TinyBanker
26. CHAOS
27. NEsha
28. Hercules
29. Gandalf
30. RDW
31. hershell
32. ARCANUS
33. braincrypt
34. Klingon RAT
</pre>
</details>
</p>

Универсальность языка позволяет скомпилировать один и тот же исходник под разные архитектуры и ос, в том числе и под линукс в виде ELF файла. 

Недавно я посмотрел видео [Golang Malware: Using the attackers force against them](https://www.youtube.com/watch?v=kwrIr8Ydwro), в котором прозвучала интересная мысль. Мысль заключалась в том, что существует возможность вычисления примерного timestamp у бинарника, используя зависимости. Алгоритм вычисления из видео следующий:

1. Берем golang ELF малварь
2. Получаем список зависимостей
3. Получаем версию зависимости
4. Получаем дату конкретной версии (релиза) зависимости
5. Создаем список из дат
6. Берем самую позднюю дату, она и будет являться примерным (минимальным) timestamp

## Код

Решено было реализовать эту мысль в коде. Для получения зависимостей использовался проект [gore](github.com/goretk/gore). Надо иметь в виду, что зависимости из бинарника возможно получить, только если у него есть секция [gopclntab](https://www.mandiant.com/resources/golang-internals-symbol-recovery). Для примера будем рассматривать образец малвари Kinsing. [Kinsing](https://attack.mitre.org/software/S0599/) - это golang малварь, которая запускает майнер. gore возвращает список зависимостей в таком виде:

<pre>
\go\pkg\mod\github.com\armon\go-socks5@v0.0.0-20160902184237-e75332964ef5
\go\pkg\mod\github.com\tklauser\go-sysconf@v0.3.10
\go\pkg\mod\github.com\shirou\gopsutil@v3.21.11+incompatible\mem
\go\pkg\mod\github.com\kelseyhightower\envconfig@v1.4.0
\go\pkg\mod\github.com\tklauser\numcpus@v0.4.0
\go\pkg\mod\github.com\shirou\gopsutil@v3.21.11+incompatible\process
\go\pkg\mod\github.com\asaskevich\govalidator@v0.0.0-20210307081110-f21760c49a8d
\go\pkg\mod\github.com\google\btree@v1.0.1
\go\pkg\mod\github.com\shirou\gopsutil@v3.21.11+incompatible\cpu
\go\pkg\mod\github.com\hashicorp\yamux@v0.0.0-20211028200310-0bc27b27de87
\go\pkg\mod\github.com\peterbourgon\diskv@v2.0.1+incompatible
\go\pkg\mod\github.com\go-resty\resty\v2@v2.7.0
\go\pkg\mod\github.com\op\go-logging@v0.0.0-20160315200505-970db520ece7
\go\pkg\mod\golang.org\x\net@v0.0.0-20220225172249-27dd8689420f\context
\go\pkg\mod\golang.org\x\net@v0.0.0-20220225172249-27dd8689420f\publicsuffix
\go\pkg\mod\github.com\nu7hatch\gouuid@v0.0.0-20131221200532-179d4d0c4d8d
\go\pkg\mod\github.com\shirou\gopsutil@v3.21.11+incompatible\host
\go\pkg\mod\github.com\shirou\gopsutil@v3.21.11+incompatible\net
\go\pkg\mod\github.com\kardianos\osext@v0.0.0-20190222173326-2bc1f35cddc0
\go\pkg\mod\github.com\paulbellamy\ratecounter@v0.2.0
\go\pkg\mod\golang.org\x\sys@v0.0.0-20220319134239-a9b59b0215f8\unix
\go\pkg\mod\github.com\shirou\gopsutil@v3.21.11+incompatible\internal\common
</pre>

Если посмотреть внимательно, обнаружится, что версии зависимостей Golang представляет в трех видах:

1. v0.0.0-20160902184237-e75332964ef5
2. v3.21.11+incompatible\cpu
3. v0.3.10

В первом виде, версия зависимости всегда v0.0.0, то есть зависимость не имеет релизов, а значит сама строка версии содержит дату создания. Поэтому мы просто берем дату из строки.

Во втором виде, `incompatible` означает, что зависимость не имеет файла `go.mod` и имеет версию больше 2. Мы отрбасываем слово incompatible и берем только версию.

В третьем виде у нас ничего нет, кроме версии, поэтому переходим к следующему шагу.

Зная только версию зависимости, мы с помощью [golang библиотеки для работы с GithubAPI](https://github.com/google/go-github) можем попробовать получить дату релиза. Рассмотрим на примере зависимости `\go\pkg\mod\github.com\tklauser\go-sysconf@v0.3.10`.
Так как не все репозитории зависимостей имеют релизы, а только тэги, используется метод [`ListTags`](https://docs.github.com/en/rest/git/tags#get-a-tag). `owner` и `repo` в данном случае будут значения - `tklauser` и `go-sysconf`

```go
tags, resp, err := client.Repositories.ListTags(context.Background(), owner, repo, nil)
```

Мы получим данные о каждом тэге (релизе)

![](/assets/images/golang_timestamp/1.png)

У тэга есть поле `commit` с полем `SHA`.  Используя Github Commit API мы получаем информацию о коммите

```go
commit, resp, err := client.Repositories.GetCommit(context.Background(), owner, repo, sha, nil)
```
В ответе будет дата коммита.

![](/assets/images/golang_timestamp/2.png)

Теперь у нас есть даты конкретных релизов конкретной зависимости. Из этих дат находим самую позднюю, она и будет примерным (минимальным) timestamp образца:

![](/assets/images/golang_timestamp/3.png)

[**Исходный код**](https://github.com/thatskriptkid/chrononz)
