---
layout: post
title: Анализ фишинговой атаки на клиентов HalykBank
tags: [windows, malware analysis]
category: [ru]
---

> Все это было рассказано на 2600 Qazaqstan в виде презентации (7.12.2018).
> Образец малвари вы можете скачать [тут](/assets/files/halybank_payload.zip), пароль: **infected**. Делайте это на свой страх и риск!

Пользователям банка [HalykBank](https://halykbank.kz/) было разослано следующее письмо с вложением, в виде .iso файла.

Заголовок:

![](/assets/images/ru//HalykBankPhishing/1-header%20email.png)

Тело письма:

![](/assets/images/ru//HalykBankPhishing/2-body%20email.png)

Если мы скачаем приложенный файл, то увидим, что он представляет собой .iso файл, а внутри лежит _Scan00987643.exe_.

![](/assets/images/ru//HalykBankPhishing/3-iso.png)

Почему именно iso файл? Возможно, злоумышленники пытались тем самым обойти детектирование при запуске в VM антивируса.

Результат virustotal: 

![](/assets/images/ru//HalykBankPhishing/4-VirusTotal.png)

Virustotal некорректно определил, что это за малварь и какое у нее кодовое имя, далее мы увидим почему. Посмотрим поближе, что за файл:

![](/assets/images/ru//HalykBankPhishing/5-DIE.png)

Exe написан на C#, обфусцирован, с помощью [.NET Reactor](https://www.eziriz.com/dotnet_reactor.htm). Снять обфускацию мы можем с помощью [de4dot](https://github.com/0xd4d/de4dot)

![](/assets/images/ru//HalykBankPhishing/6-de4dot%20description.png)

![](/assets/images/ru//HalykBankPhishing/6-de4dot%20console.png)

Отлично! Откроем очищенный файл в hex'е, и посмотрим глазами на предмет странностей и находим такой паттерн

![](/assets/images/ru//HalykBankPhishing/7-in%20hex%20editor.png)

Больше всего похоже на заксоренные данные. Больше ничего интересного при статическом анализе найдено не было, начнем дебажить! И поможет нам в этом [dnSpy](https://github.com/0xd4d/dnSpy)! Это дебаггер и .NET декомпилятор. С помощью него, можно редактировать exe-шники на C# и дебажить их, не имя исходного кода. Дебажить можно любые приложения, написанные с помощью .NET Framework, .NET Core, Unity. А самое главное - можно редактировать код на ходу (во время выполнения)! Открыв exe в dnSpy, мы видим, что он деобфусцировался не полностью

![](/assets/images/ru//HalykBankPhishing/9-obfuscate.png)

Почитав код в entry point, мы можем увидеть, что малварь подгружает какой-то дополнительный, вредоносный payload

![](/assets/images/ru//HalykBankPhishing/10-loading%20payload.png)

Assembly.Load() принимает байтовый массив с вредоносным пейлодом. DnSpy позволяет нам редактировать EXE

![](/assets/images/ru//HalykBankPhishing/11-dnsPy%20edit.png)

Меняем загрузку пейлоада на сохранение его в файл

![](/assets/images/ru//HalykBankPhishing/12-editing%20function.png)

Ура! Мы выгрузили первый пейлоад!

![](/assets/images/ru//HalykBankPhishing/13-extracted%20first%20payload.png)

Это тоже C# и тоже обфусцирован (DIE этого не определил) *(

![](/assets/images/ru//HalykBankPhishing/14-DIE%20on%20payload.png)

Итого, на данный момент: ISO файл, в котором обфусцированный EXE, который загружает еще один EXE, который тоже обфусцирован, который ...

Опять натравливаем de4dot

![](/assets/images/ru//HalykBankPhishing/16-de4dot%20second%20payload.png)

Загружаем полученный exe в dnSpy и снова читаем исходники. Видим, что все строки в малваре закодированы и раскодируются в рантайме

![](/assets/images/ru//HalykBankPhishing/17-encoded%20strings-1.png)

![](/assets/images/ru//HalykBankPhishing/17-encoded%20strings-2.png)

![](/assets/images/ru//HalykBankPhishing/17-encoded%20strings-3.png)

Ресурс с закодированными строками. По виду это base64

![](/assets/images/ru//HalykBankPhishing/18-strings%20in%20resources.png)

Пейлоад копирует себя в папку Users под именем “null” и добавляется в автозагрузку

![](/assets/images/ru//HalykBankPhishing/19-payload%20actions-1.png)

![](/assets/images/ru//HalykBankPhishing/19-payload%20actions-2.png)

Из ресурсов подгружается dll, которой, в качестве параметра передается раскодированный exe из ресурсов

![](/assets/images/ru//HalykBankPhishing/20-load%20DLL.png)

![](/assets/images/ru//HalykBankPhishing/20-load%20DLL-2.png)

Этот exe предстовляет собой еще один обфусцрованый exe, со странным именем Reborn stub.

![](/assets/images/ru//HalykBankPhishing/26-reborn%20stub.png)

Ранее, я о нем где-то слышал и поэтому решил для начала загуглить

![](/assets/images/ru//HalykBankPhishing/27-reborn%20stub%20on%20google.png)

Оказалось, что это пейлоад популярной малвари Hawkeye keylogger. Решено было не анализировать дальше вручную, а воспользоваться сервисом [any.run](https://any.run/), для дальнейшего динамического анализа

![](/assets/images/ru//HalykBankPhishing/28-any%20run.png)

Цепочка работы RebornStub.exe

![](/assets/images/ru//HalykBankPhishing/29-any%20run%202.png)

Это Hawkeye Keylogger! Если вы помните, изначально Virustotal так его не определял 

![](/assets/images/ru//HalykBankPhishing/30-any%20run%203.png)

Коннектится наружу!

![](/assets/images/ru//HalykBankPhishing/31-any%20run%204.png)

Грузит dll мозилы, а значит крадает пароли фаерфокса!

![](/assets/images/ru//HalykBankPhishing/32-any%20run%205.png)

Ну и общая схема

![](/assets/images/ru//HalykBankPhishing/33-any%20run%206.png)