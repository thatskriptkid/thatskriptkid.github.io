---
layout: post
title: Советы для исследования .NET малвари
tags: [note, malware-analysis, windows]
category: [ru]
---

 У нас есть малварь, которая декодирует ресурс DE. 

![Alt text](/assets/images/ru/dotnet_lifehacks/01.png)

С помощью скрипта [stego](https://github.com/thatskriptkid/malware-analysis-tools/tree/master/dotnet/stego) можно расшифровать картинку.

![Alt text](/assets/images/ru/dotnet_lifehacks/02.png)

Результат декодирования:

![Alt text](/assets/images/ru/dotnet_lifehacks/03.png)

Также этот скрипт способен производить обратную операцию - превращать файл в стеганографическое изображение. Может использоваться в сценарии, когда вы пропатчили вредоносный файл, и хотите обратно закодировать его в картинку, чтобы малварь в процессе своей работы расшифровала ваш пропатченный бинарник. Не забудьте указать ширину и высоту картинки, как в оригинале.

![Alt text](/assets/images/ru/dotnet_lifehacks/04.png)

## Дамп пейлодов 

1. Часто .net малварь выполняет несколько стадий распаковки, задействуя несколько пейлодов. Чтобы просмотреть вредоносные пейлоды в памяти можно воспользоваться инстурментом ProcessHacker:

![](/assets/images/ru/dotnet_lifehacks/net_assemblies.png)

2. Дампнуть отдельные .NET Assembly из памяти можно тулзой [ExtremeDumper](https://github.com/wwh1004/ExtremeDumper). hollows_hunter может не видеть все пейлоды в памяти.

3. Также можно сделать дамп, поставив брейкпоинт на загрузку модулей в dnSpy. Способ показан в [видео](https://youtu.be/tenNFzM-MM0?si=15ko7638EPtNYa5i&t=817). Нам надо отобразить окно Модули.

![](/assets/images/ru/dotnet_lifehacks/dump_1.png)

Затем поставить "*" в имена отслеживаемых модулей.

![](/assets/images/ru/dotnet_lifehacks/dump_2.png)

Запускаем малварь и мы увидим все загружаемые модули.

![](/assets/images/ru/dotnet_lifehacks/dump_3.png)

## Вызов отдельных методов

### Способ первый

Малварь может содержать функции по расшифровки/декодированию строк или другие интересные функции, которые хочется вызвать отдельно (только их). Например, малварь содержит функцию по декодированию строки `CausalitySource`

![](/assets/images/ru/dotnet_lifehacks/mal_func.png)

Аргументы, которая она принимает:

![](/assets/images/ru/dotnet_lifehacks/args.png)

Существует два удобных способа вызвать эту функцию с нужными аргументами. Первый - с помощью LinqPad

![](/assets/images/ru/dotnet_lifehacks/linqpad_descr.png)

[Linqpad](https://www.linqpad.net/Download.aspx) - это легковесный исполнитель dotnet кода, которым удобно пользоваться в виртуалке. Можно скачивать 5 версию, так как .NET framework по умолчанию стоит в Windows 7. Теперь мы можем просто скопировать из образца код, вставить его в редактор и исполнить.

![](/assets/images/ru/dotnet_lifehacks/Linqpad.png)

Необязательно писать полноценную программу с Main() и т.д., можно использовать просто куски кода, для этого в выпадающем меню выбираем `Expression` или `Statement`.

![](/assets/images/ru/dotnet_lifehacks/linq_menu.png)

Например, исполнение отдельных строк кода:

![](/assets/images/ru/dotnet_lifehacks/linq_statement.png)

### Способ второй

Часто образец малвари в своих методах использует собственные константы, другие методы и т.д. и переносить кучу кода/переменные в Linqpad нерелевантно. Поэтому можно воспользоваться втором способом - powershell. Ничего устанавливать не нужно, powershell по умолчанию есть в Windows 7.  Запускаем нужный экземпляр powershell, чтобы его разрядность совпадала с разрядностью образца.  Первым делом нам необходимо подгрузить наш вредоносный Assembly командой:

```
$malware = [Reflection.Assembly]::LoadFile("path")
```

Также это можно сделать с помощью cmdlet (более современный способ). В этом случае файл должен иметь расширение `.dll`:

```
Add-Type -Path foo.dll
```

![](/assets/images/ru/dotnet_lifehacks/ps_3.png)

Убедиться что операция завершена успешна поможет команда просмотра списка всех загруженных Assembly:

```
[appdomain]::currentdomain.GetAssemblies()
```

![](/assets/images/ru/dotnet_lifehacks/ps_1.png)

Чтобы вызвать конкретный, **статический** метод мы используем следующую конструкцию:

```
[имя_namespace.имя_класса]::имя_функции("аргументы")
```

Например:

![](/assets/images/ru/dotnet_lifehacks/ps_2.png)

Если метод НЕ статический, то в начале инстанцируем класс:

```
$instance = New-Object имя_namespace.имя_класса
$result = $instance.имя_функции()
echo $result
```

Пример вызова нестатического метода:

![](/assets/images/ru/dotnet_lifehacks/ps_4.png)

#### Полезные powershell команды

*ассембли (Assembly) - любой .NET EXE или DLL

Стандартный размер шрифта powershell очень мал. Увеличить его можно кликнув правой кнопкой на иконку - Свойства.

![](/assets/images/ru/dotnet_lifehacks/ps_font.png)

Очистка окна консоли:

```
Clear-Host
```

Если для каких-либо операций нам необходимо подгрузить зависимость:

```
[System.Reflection.Assembly]::LoadWithPartialName("System.Drawing")
```

Например, подгрузив зависимость `Drawing` мы можем использовать класс `Bitmap`:

```
$bmp = New-Object System.Drawing.Bitmap("D:\\path")
```

Вывод всех типов внутри бинарника:

```
$malware.gettypes()
```

![](/assets/images/ru/dotnet_lifehacks/ps_types.png)

Поиск по выводу какой-либо команды:

```
| Select-String -Pattern "строка_для_поиска"
```

![](/assets/images/ru/dotnet_lifehacks/ps_search.png)

Вывод только публичных классов:

```
$assembly.GetTypes() | ? {$_.IsClass -and $_.IsPublic}
```

![](/assets/images/ru/dotnet_lifehacks/ps_is.png)

Вывод всех текущих загруженных Assembly в отсортированном виде (также эта команда может пригодится или нет при форензике):

```
[appdomain]::currentdomain.getassemblies() | sort -property fullname | format-table fullname
```

![](/assets/images/ru/dotnet_lifehacks/ps_5.png)

Дамп захардкоженных переменных ассембли. Допустим у нас есть малварь, которая содержит некий статический байтовый блоб. 

![](/assets/images/ru/dotnet_lifehacks/byte_blob.png)

Если вы хотите дампнуть его, это можно сделать двумя способами. Динамически - запустить ассембли в дебаггере и дампнуть значение в рантайме. Статически - дампнуть, используя powershell. Чтобы дампнуть используя powershell мы подгружаем ассембли и сохраняем в файл желаемую переменную.

```
$malware = [Reflection.Assembly]::LoadFrom("path") # подгружаем наш вредоносный ассембли
[io.file]::WriteAllBytes('path_to_save_var', [namespace.class]::var_name) # сохраняем в файл
```

Пример:

![](/assets/images/ru/dotnet_lifehacks/save_var.png)

Если требуемая переменная НЕстатическая, то просто создайте инстанс соответствующего класса.

Если вам лень постоянно вводить команды для дампа, то можно написать скрипт. Чтобы скрипт работал, не забудьте включить исполнение скриптов на системе:

```
set-executionpolicy remotesigned
```

### Лайфхаки

3. AgentTesla содержит пейлоды в виде ресурсов. Один из пейлодов является стеганографической картинкой. Если мы хотим пропатчить пейлод, который распаковывается на какой-либо стадии, нам необходимо:
   1. Преобразовать ресурс из картинки в файл
   2. Пропатчить
   3. Преобразовать файл обратно в картинку
   4. Заменить ресурс картинку
В этом может помочь скрипт [stego](https://github.com/thatskriptkid/malware-analysis-tools/tree/master/dotnet/stego). 

1. Например, мы хотим написать YARA правило на кусок .NET кода. 
   
![](/assets/images/ru/dotnet_lifehacks/yara.png)

*[вспомогательная ссылка](https://www.binarydefense.com/resources/blog/creating-yara-rules-based-on-code/)

*[список опкодов](https://en.wikipedia.org/wiki/List_of_CIL_instructions)

5. LinqPad поддерживает работу с файлами

![](/assets/images/ru/dotnet_lifehacks/linqpad_files.png)

6. Иногда, .NET малварь может содержать GUID. Это поле идентифицрует конкретный проект на компе разработчика. Соответственно, если у нескольких отдельных образцов один и тот же GUID, то с уверенностью можно говорить, что они были разработаны на одном компе и скорей всего одним и тем же разработчиком.

![](/assets/images/ru/dotnet_lifehacks/guid.png)

7. Если нам попался .NET Bundle, то дампнуть файлы бандла можно с помощью AsmResolver и powershell:

```
[Reflection.Assembly]::LoadFrom("C:\AsmResolver\AsmResolver.DotNet.dll") | Out-Null
$extractionPath = "C:\Extracted\"
$manifest = [AsmResolver.DotNet.Bundles.BundleManifest]::FromFile("C:\RiotClientServices.exe")

foreach($file in $manifest.Files)
{
    $fileInfo = [IO.FileInfo]::new($extractionPath + $file.RelativePath)
    $fileInfo.Directory.Create()
    [IO.File]::WriteAllBytes($fileInfo.FullName, $file.getData($true))
}
```
[Более подробная информация](https://research.checkpoint.com/2023/byos-bundle-your-own-stealer/)

8. Библиотека [AsmResolver](https://github.com/Washi1337/AsmResolver), с помощью которой можно модифицировать .net бинарники. Например, удалять ненужные методы:

```
[Reflection.Assembly]::LoadFrom("C:\AsmResolver\AsmResolver.DotNet.dll") | Out-Null
$obfuscated = "C:\RiotClientServices.dll"
$moduleDef = [AsmResolver.DotNet.ModuleDefinition]::FromFile($obfuscated)
# Removing junk methods
foreach($type in $moduleDef.GetAllTypes())
{
    foreach($method in [array]$type.Methods.Where{$_.HasMethodBody})
    {
        if(($method.MethodBody.Instructions.Where{$_.Opcode.Mnemonic -like "call" -and
            $_.Operand.FullName -like "*System.Console::WriteLine*"}).count -eq  5)
        {
            $type.Methods.Remove($method) | Out-Null
        }
    }
}
```

9. Если надо отдебажить библиотеку .NET, то можно воспользоваться [SharpDllLoader](https://github.com/hexfati/SharpDllLoader)

10. Образцы можно группировать по TypeRef hash. Вычислить его поможет [скрипт в гитлаб](https://github.com/thatskriptkid/malware-analysis-tools/tree/master/dotnet/type_ref_hasher). TypeRef hash лучше использовать отсортированный и включая сущности, которые ссылаются сами на себя (sorted, include self-referenced entries).

11. Если функция расшифровки строк малвари вызывается много раз, можно написать следующий скрипт (пример):

```
$malware = [Reflection.Assembly]::LoadFile("C:\my\f.dll")
[jKcEiEe]::TFkR(980214767)| Out-File -FilePath c:\my\output.txt
[jKcEiEe]::TFkR(980214689) | Add-Content C:\my\output.txt
[jKcEiEe]::TFkR(980214350) | Add-Content C:\my\output.txt
[jKcEiEe]::TFkR(980214730) | Add-Content C:\my\output.txt
[jKcEiEe]::TFkR(980214391) | Add-Content C:\my\output.txt
[jKcEiEe]::TFkR(980214713) | Add-Content C:\my\output.txt
[jKcEiEe]::TFkR(980214747) | Add-Content C:\my\output.txt
[jKcEiEe]::TFkR(980214670) | Add-Content C:\my\output.txt
[jKcEiEe]::TFkR(980214364) | Add-Content C:\my\output.txt
[jKcEiEe]::TFkR(980214543) | Add-Content C:\my\output.txt 
```

12. Иногда малварь для сокрытия пейлода использует [Fody](https://github.com/Fody/Home/)/[Costura](https://github.com/Fody/Costura). Costura предназначена для добавления зависимостей/ресурсов в единный бинарник. Полезные ссылки по таким кейсам:
    
[example-analysis-of-multi-component-malware](https://www.cyren.com/blog/articles/example-analysis-of-multi-component-malware)

[play-ransomware-volume-shadow-copy](https://symantec-enterprise-blogs.security.com/blogs/threat-intelligence/play-ransomware-volume-shadow-copy)

13. Получение данных о X509 сертификате кодом в LINQPAD. Также можно проверить валидность файла:

```

void Main()
{
	string fileName = "c:\\my\\OUTLOOK.CFG";
	string text2 = "test-908@testgmail-398504.iam.gserviceaccount.com";
	X509Certificate2 x509 = new X509Certificate2(fileName, "notasecret", X509KeyStorageFlags.Exportable);
	
	Console.WriteLine("{0}Subject: {1}{0}", Environment.NewLine, x509.Subject);
    Console.WriteLine("{0}Issuer: {1}{0}", Environment.NewLine, x509.Issuer);
    Console.WriteLine("{0}Version: {1}{0}", Environment.NewLine, x509.Version);
    Console.WriteLine("{0}Valid Date: {1}{0}", Environment.NewLine, x509.NotBefore);
    Console.WriteLine("{0}Expiry Date: {1}{0}", Environment.NewLine, x509.NotAfter);
    Console.WriteLine("{0}Thumbprint: {1}{0}", Environment.NewLine, x509.Thumbprint);
    Console.WriteLine("{0}Serial Number: {1}{0}", Environment.NewLine, x509.SerialNumber);
    Console.WriteLine("{0}Friendly Name: {1}{0}", Environment.NewLine, x509.PublicKey.Oid.FriendlyName);
    Console.WriteLine("{0}Public Key Format: {1}{0}", Environment.NewLine, x509.PublicKey.EncodedKeyValue.Format(true));
    Console.WriteLine("{0}Raw Data Length: {1}{0}", Environment.NewLine, x509.RawData.Length);
    Console.WriteLine("{0}Certificate to string: {1}{0}", Environment.NewLine, x509.ToString(true));
    Console.WriteLine("{0}Certificate to XML String: {1}{0}", Environment.NewLine, x509.PublicKey.Key.ToXmlString(false));
}

// Define other methods and classes here

```
    
## Быстрая разработка на .NET

Иногда необходимо по-быстрому полностью реализовать какой-нибудь алгоритм/написать код. Например, повторить кусок кода из малвари, который работает с файлами, открывает их, сохраняет и т.д.. 

1. Скачиваем [VSCode](https://code.visualstudio.com/)
2. Устанавливаем два расширения: [C# Dev Kit](https://marketplace.visualstudio.com/items?itemName=ms-dotnettools.csdevkit) и [C#](https://marketplace.visualstudio.com/items?itemName=ms-dotnettools.csharp)
3. Скачиваем [dotnet sdk](https://dotnet.microsoft.com/en-us/download)
4. Чтобы убедиться, что dotnet установлен и работает корректно в командной строке выполняем команду `dotnet`
5. Открываем `VSCode` через `Developer Command Prompt` командой `code`

![](/assets/images/ru/dotnet_lifehacks/cmd_prompt.png)

6. Создаем новый проект типа Console командой:

```
dotnet new console
```

7. Если не хватает какой-либо зависимости:

```
dotnet add package System.Drawing.Common --version 7.0.0
```

8. Собрать проект

```
dotnet clear
dotnet build
```

9.  Чтобы получить бинарник пригодный для запуска на другом компе (не забудьте указать нужную архитектуру):

```
dotnet publish -r win7-x64 
```

Ссылки по этой теме:

https://learn.microsoft.com/en-us/dotnet/core/rid-catalog

https://learn.microsoft.com/en-us/dotnet/core/tools/dotnet-build

https://learn.microsoft.com/en-us/dotnet/core/deploying/

Publishing your app as self-contained produces an application that includes the .NET runtime and libraries, and your application and its dependencies. Users of the application can run it on a machine that doesn't have the .NET runtime installed.

10. Чтобы бинарник работал на другой компе или в виртуалке, надо скопировать ВСЮ папку `bin\x64\Debug\net7.0\win7-x64\publish`
