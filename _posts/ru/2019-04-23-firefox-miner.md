---
layout: post
title: Анализ майнера FirefoxUpdate.exe
tags: [windows, malware analysis]
category: [ru]
---

MD5:342E134B3DE901EE6A915909AECDAC4A

EXE ничем не упакован, написан на C#. Два антивируса на Virustotal определяют его, как "Win32.Trojan.WisdomEyes" и "UDS:DangerousObject.Multi.Generic".

![Virustotal](/assets/images/ru//FirefoxUpdate/imgs/virustotal.png)

Судя по пути хранения оталдочных символов, имя пользователя на компе разработчика - Achraf

![pdb_path](/assets/images/ru//FirefoxUpdate/imgs/PDB_path.png)

  При запуске, происходит скачивание страницы по адресу http://164.132.197.47/security-updates/available.php, в которой хранится статус доступности сервера. 

![AvailableChecking](/assets/images/ru//FirefoxUpdate/imgs/screenshot-from-2018-05-19-22-.png)

Если страница содержит строку "available", то значит сервер доступен. Иначе, отображается окно, с текстом
"Unable to update, please try later"

![UnableToUpdate](/assets/images/ru//FirefoxUpdate/imgs/Unable_to_update.png)

Далее создается папка "C:/Program Files/ChromeUpdates/". В зависимости от разрядности системы, к адресу "http://164.132.197.47/security-updates/" добавляется 32 или 64, и по нему скачивается три файла: update.exe, config.json, ChromePassBackup.exe. Файлы сохраняются в папку "C:/Program Files/ChromeUpdates/".

![ChromeUpdateFolder](/assets/images/ru//FirefoxUpdate/imgs/ChromeUpdatesFolder.png)

Содержимое файла config.json:

![Config json](/assets/images/ru//FirefoxUpdate/imgs/Config-json.png)

Судя по url'ам, можно понять, что это конфиг Monero майнера.

![MoneroSite](/assets/images/ru//FirefoxUpdate/imgs/Monero_site.png)


По окончанию скачивания, создается новый процесс cmd.exe, со скрытым окном, со следующими аргументами:

```
/C schtasks /create /tn SecurityUpdates /tr \" \\\"C:\\Program Files\\ChromeUpdates\\update.exe\\\" \"  /sc onstart /RU SYSTEM
```

Из командной строки, запускается ```schtasks.exe```. С помощью этой программы создается (```/create```) запланированная задача, с именем (```/tn```) SecurityUpdate, которая будет запускать (```/tr```) файл ```C:\\Program Files\\ChromeUpdates\\update.exe\\\```, при старте системы (```/sc onstart```), используя пользователя ```SYSTEM``` (```/RU SYSTEM```).

![schtask](/assets/images/ru//FirefoxUpdate/imgs/schtasks.png)

Далее генерируется строка, длинной 12 символов, состоящая из случайных символов строки ```abcdefghijklmnopqrstuvwxyz0123456789```.
Эта строка подставляется вместо NULL в ```"\"worker-id\"```: null", в файле ```"C:\\Program Files\\ChromeUpdates\\config.json"```. Похоже на случайный токен для майнинг клиента.

![WorkerIdUrl](/assets/images/ru//FirefoxUpdate/imgs/Worker-id-url.png)

Далее создается новый процесс cmd.exe с аргументами 

```"/C netsh interface ipv4 add dnsservers ИМЯ_НЕЛОКАЛЬНОГО_СЕТЕВОГО_ИНТЕРФЕЙСА address=1.1.1.1 index=1"```

Данный процесс, с помощью командной строки, добавляет dns сервер 1.1.1.1, первым по списку, ВО ВСЕ имеющиеся сетевые интерфейсы, кроме Loopback. 

![netsh](/assets/images/ru//FirefoxUpdate/imgs/netsh_dns.png)

По окончанию процесса, появляется окно "Update complete".

![UpdateComplete](/assets/images/ru//FirefoxUpdate/imgs/Update_Complite.png)

## ChromePassBackup.exe

Скачанный на предыдущем шаге exe, является утилитой, для просмотра паролей, сохраненных в Google Chrome [nirsoft](https://www.nirsoft.net/utils/chromepass.html).

## Update.exe

Этот файл был поставлен на выполнение, при каждом запуске, как запланированная задача. Virustotal определяет файл, как майнер. Репозиторий майнера: https://github.com/xmrig/xmrig/releases

![VirusTotalUpdate](/assets/images/ru//FirefoxUpdate/imgs/Update_EXE_Virustotal.png)

## Восстановление системы

1) Удалить папку ```"C:/Program Files/ChromeUpdates/"```

2) Заблокировать в фаерволе ip "hxxp://164(.)132(.)197(.)47/"

3) Удалить запланированную задачу SecurityUpdate командой:

```schtasks /Delete /TN SecurityUpdate```

4) Изменить DNS сервер на желаемый, вместо 1.1.1.1
