---
layout: post
title: Детектирование антивирусов с помощью перечисления имен мини-фильтров
category: [translations]
tag: [windows, malware]
---

![alt_text](/assets/images/translations/detectav/image-000.jpg)

_[Оригинал статьи](https://vxug.fakedoma.in/papers/VXUG/Exclusive/IdentifyingAntivirusSoftwarebyenumeratingMinifilterStringNames.pdf)_

_Автор оригинала: [smelly__vx](https://twitter.com/smelly__vx)_

**Введение:**

В последнее время я подробно изучал внутреннюю работу антивирусов. У меня накопилось много информации, которую я собираюсь задокументировать и поделиться. Но для начала я решил написать небольшую статью об определении антивирусов по соответствующим [мини-фильтрам](https://docs.microsoft.com/en-us/windows/win32/api/fltuser/nf-fltuser-filterconnectcommunicationport#parameters). Я лично не видел, чтобы кто-нибудь описывал данную технику, хотя она является очень простой. К сожалению, как и всегда, эта техника имеет некоторые ограничения, и они должны быть проговорены заранее:

1. Требуются права администратора
2. Код, реализующий технику, возвращает имя мини-фильтра антивируса. Это означает, что вам необходимо самим создать список известных имен. Создать список достаточно просто, но придется хардкодить строки, что печально. Для создания списка можете взглянуть на список уникальных идентификаторов (altitudes) мини-фильтров в [MSDN](https://docs.microsoft.com/en-us/windows-hardware/drivers/ifs/allocated-altitudes).
3. Для любопытных: в представленном далее коде вызывается API в NTDLL. Его можно переписать на использование системных вызовов, что является более трудным путем, но все еще возможным.

В любом случае, я надеюсь вам понравится эта прикольная техника.

-smelly

**Кратко о мини-фильтрах**

Несмотря на то, что существует возможность обойти антивирусные хуки в пользовательском пространстве (например [HellsGate](https://github.com/vxunderground/VXUG-Papers/tree/main/Hells%20Gate)), всё ещё остаётся последнее серьезное препятствие: мини-фильтры. Система мини-фильтров обычно используется для анализа поведения программ, в частности детектирования действий программ-вымогателей. Она является сильной стороной антивирусов.

Использование мини-фильтров - это де-факто стандартная техника антивирусов. В серии статей [Antivirus Artifacts](https://vxug.fakedoma.in/papers/VXUG/Mirrors/ANTIVIRUS_ARTIFACTS_III.pdf) вы найдете упоминание мини-фильтров для каждого популярного антивируса (а также для EDR!). По своей сути, мини-фильтры позволяют получить полную власть над файловой системой NTFS. С помощью них антивирусы  могут выполнять действия перед и после запуска программы, а также во время выполнения программы.

Короче говоря, вы можете прочитать все о мини-фильтрах и их использовании в замечательной статье [An Introduction to Standard and Isolation Minifilters](https://www.osr.com/nt-insider/2017-issue2/introduction-standard-isolation-minifilters/) (для которой тоже есть[ наш перевод](https://vx-underground.org/ru/translations/windows_vx/MinifiltersIntro.pdf)! ). У мини-фильтров много других применений, помимо использования антивирусами.  Если вам интересно, как антивирусы работают с мини-фильтрами с технической точки зрения, вы можете скачать и посмотреть примеры с [MSDN](https://docs.microsoft.com/en-us/windows-hardware/drivers/samples/file-system-driver-samples), они достойны внимания.

**API мини-фильтров для пользовательского пространства (usermode)**

Windows предоставляет API для взаимодействия с мини-фильтрами. Оно позволяет антивирусам, работающим в пользовательском пространстве, взаимодействовать с мини-фильтрами, которые работают на уровне ядра. Взаимодействие характеризуется двумя функциями:

1. [FilterConnectCommunicationPort](https://docs.microsoft.com/en-us/windows/win32/api/fltuser/nf-fltuser-filterconnectcommunicationport)
2. [DeviceIoControl](https://docs.microsoft.com/en-us/windows/win32/api/ioapiset/nf-ioapiset-deviceiocontrol)

Некоторые антивирусы используют функции в [fltuser.h](https://docs.microsoft.com/en-us/windows/win32/api/fltuser/) для работы с мини-фильтрами. В то же время, другие антивирусы могут взаимодействовать напрямую, с помощью DeviceIoControl. Объяснение причин такого поведения выходит за рамки статьи. Тем не менее, суть в том, что такой API существует и может использоваться в пользовательском пространстве. Единственным сходством является то, что функции FilterConnectCommunicationPort и DeviceIoControl требуют [повышенных привилегий](https://docs.microsoft.com/en-us/windows/win32/secauthz/mandatory-integrity-control). Иначе никак.

**Перечисление имен мини-фильтров**

В отличии от традиционного понятия “порт”, который обычно представляется нам беззнаковым 16 битным числом (1 - 65535), порты мини-фильтра в Windows используют WCHAR строки, в качестве уникальных идентификаторов. Это можно увидеть в описании параметра lpPortName функции FilterConnectCommunicationPort:

_Указатель на WCHAR строку, содержащую имя порта (например  L"\MyFilterPort")_

Тем самым, каждый мини-фильтр имеет имя порта, которое мы можем найти функциями [FilterFindFirst](https://docs.microsoft.com/en-us/windows/win32/api/fltuser/nf-fltuser-filterfindfirst), [FilterFindNext ](https://docs.microsoft.com/en-us/windows/win32/api/fltuser/nf-fltuser-filterfindnext)и [FilterFindClose](https://docs.microsoft.com/en-us/windows/win32/api/fltuser/nf-fltuser-filterfindclose). Это похоже на перечисление файлов функциями [FindFileFirst ](https://docs.microsoft.com/en-us/windows/win32/api/fileapi/nf-fileapi-findfirstfilea), [FindNextFile ](https://docs.microsoft.com/en-us/windows/win32/api/fileapi/nf-fileapi-findnextfilea)и [FindClose](https://docs.microsoft.com/en-us/windows/win32/api/fileapi/nf-fileapi-findclose).

**Заключение**

После изучения кода, пожалуйста изучите внутреннюю работу упомянутых API функций. Они очень интересны и если вы исследователь/Red Team, после данной статьи вы можете провалиться в еще более глубокую кроличью нору. Она покажет вам неизвестную часть Windows. Вы можете свободно связаться со мной, если что-то найдете. Это реально крутые вещи.

**Код**

1. До вызова любых API мы должны создать структуру [FILTER_FULL_INFORMATION](https://docs.microsoft.com/en-us/windows-hardware/drivers/ddi/fltuserstructures/ns-fltuserstructures-_filter_full_information). Когда вы вызываете FilterFindFirst, параметр lpBytesReturned получает количество байт, в виде DWORD, в буфере с FILTER_FULL_INFORMATION. Традиционно, мы сначала вызываем FilterFindFirst с параметром dwBufferSize равным 0, для того чтобы узнать сколько памяти необходимо выделить. Однако, в моих предварительных тестах я обнаружил, что размер зависит от имени строки. Поэтому я использую [MAX_PATH (260 байт)](https://docs.microsoft.com/en-us/windows/win32/fileio/naming-a-file?redirectedfrom=MSDN#paths). Это значение более чем достаточно для имени мини-фильтра.

```c++
DWORD dwError = ERROR_SUCCESS, dwBufferSize = 0 ;
HRESULT Result;
HANDLE Filter = INVALID_HANDLE_VALUE, ProcessHeap = GetProcessHeap();
PFILTER_FULL_INFORMATION FilterInformation = NULL ;

FilterInformation = (PFILTER_FULL_INFORMATION)HeapAlloc(ProcessHeap, HEAP_ZERO_MEMORY, MAX_PATH);

if (FilterInformation == NULL )
    goto FAILURE;

Result = FilterFindFirst(FilterFullInformation, FilterInformation, MAX_PATH, &dwBufferSize, &Filter);

if (Result != S_OK || Filter == INVALID_HANDLE_VALUE)
{
    SetLastError(Win32FromHResult(Result));
    goto FAILURE;
}

_putws(FilterInformation->FilterNameBuffer);
```


После первого вызова FilterFindFirst мы выводим FilterNameBuffer из буфера FILTER_FULL_INFORMATION (переменная FilterInformation). Обратите внимание на то, что последний аргумент, HANDLE Filter, возвращается функцией FilterFindFirst. Он будет использоваться для перечисления.

2. Второй кусок кода очень простой. Как только мы успешно получили HANDLE для перечисления, мы просто запускаем бесконечный цикл.  Если FilterFindNext вернет [ERROR_NO_MORE_ITEMS (259, 0x103)](https://docs.microsoft.com/en-us/windows/win32/debug/system-error-codes--0-499-), то мы успешно перечислили все строки мини-фильтра. Иначе, мы продолжаем выводить FilterNameBuffer. Когда мы выйдем из цикла, если наш HANDLE все еще валиден, мы закрываем его функцией FilterFindClose и как обычно, освобождаем память.  

```c++
for (;;)
{
    ZeroMemory(FilterInformation, dwBufferSize);
    Result = FilterFindNext(Filter, FilterFullInformation,        FilterInformation, MAX_PATH, &dwBufferSize);

if (Result != S_OK || Filter == INVALID_HANDLE_VALUE)
{
    if (Win32FromHResult(Result) == ERROR_NO_MORE_ITEMS)
        break ;
    SetLastError(Win32FromHResult(Result));
    goto FAILURE;
}

_putws(FilterInformation->FilterNameBuffer);

}

if (Filter)
    FilterFindClose(Filter);

if (FilterInformation)
    HeapFree(ProcessHeap, HEAP_ZERO_MEMORY, FilterInformation);

return ERROR_SUCCESS;
```

Данный API не возвращает ошибку в виде DWORD, вместо этого возвращается [HRESULT](https://docs.microsoft.com/en-us/openspecs/windows_protocols/ms-erref/705fb797-2175-4a90-b5a3-3918024b10b8). Для обработки такого типа результата я написал небольшую функцию Win32FromHResult. В ней нет ничего особенного.


```c++
DWORD Win32FromHResult(HRESULT Result)
{
    if ((Result & 0xFFFF0000 ) == MAKE_HRESULT(SEVERITY_ERROR,  FACILITY_WIN32, 0 ))
        return HRESULT_CODE(Result);

    if (Result == S_OK)
        return ERROR_SUCCESS;

    return ERROR_CAN_NOT_COMPLETE;
}
```

**Результат работы**


```
bindflt
WdFilter
storqosflt
wcifs
CldFlt
FileCrypt
luafv
npsvctrig
Wof
FileInfo
```

Это все строки порта мини-фильтра на моем компьютере. Вы можете найти их  упоминание в [MSDN](https://docs.microsoft.com/en-us/windows-hardware/drivers/ifs/allocated-altitudes). Мой друг, у которого установлен AVAST получил следующий результат: 

![](/assets/images/translations/detectav/image-001.jpg)

aswSP, aswMonFlt, и aswSnx - строки мини-фильтра

**Кроличья нора**

Функция FilterConnectCommunicationPort вызывает [NTCreateFile ](https://docs.microsoft.com/en-us/windows/win32/api/winternl/nf-winternl-ntcreatefile)с параметром в виде символьной ссылки _\\Global??\\FltMgrMsg_, которая указывает на _\FileSystem\Filters\FltMgrMsg_. Строка, которая указывается в функции FilterConnectCommunicationPort, передается в параметре Extended Attributes в NTCreateFile. 

Кусок кода из IDA:


```cpp
RtlInitUnicodeString(&DestinationString, lpcwpPortName);
v26 = DestinationString.Buffer;
v25[ 0 ] = DestinationString.Length;
v25[ 1 ] = DestinationString.MaximumLength;
*(_QWORD *)&EaBuffer[v13 + 9 ] = &DestinationString;
*(_QWORD *)&EaBuffer[v13 + 17 ] = v25;
*(_WORD *)&EaBuffer[v13 + 25 ] = wSizeOfContext;
if ( wSizeOfContext )
  memcpy_0(&EaBuffer[v13 + 33 ], Src, wSizeOfContext);
RtlInitUnicodeString(&v23, L"\\Global??\\FltMgrMsg" );
ObjectAttributes.Length = 48 ;
v14 = 64 ;
ObjectAttributes.Attributes = 64 ;
ObjectAttributes.ObjectName = &v23;
ObjectAttributes.RootDirectory = 0 i64;
*(_OWORD *)&ObjectAttributes.SecurityDescriptor = 0 i64;
if ( lpSecurityAttributes )
{
  v15 = !lpSecurityAttributes->bInheritHandle;
  ObjectAttributes.SecurityDescriptor =  lpSecurityAttributes->lpSecurityDescriptor;
  if ( !v15 )
    v14 = 66 ;
  ObjectAttributes.Attributes = v14;
}
v16 = NtCreateFile(
        &FileHandle,
        0x100003 u,
        &ObjectAttributes,
        &IoStatusBlock,
        0 i64,
        0 ,
        0 ,
        3u ,
        32 * (v8 & 1 ),
        EaBuffer,
        ( unsigned __int16)(wSizeOfContext + 24 ) + 19 );
```


В то же время, FilterFindFirst и FilterFindNext взаимодействуют с NTDLL и вызывают внутреннюю функцию _FilterpDeviceIoControl_, которая вызывает кучу других низкоуровневых функций, таких как [NtDeviceIoControlFile ](https://docs.microsoft.com/en-us/windows/win32/api/winternl/nf-winternl-ntdeviceiocontrolfile)and [NtFsControlFile](https://docs.microsoft.com/en-us/windows-hardware/drivers/ddi/ntifs/nf-ntifs-ntfscontrolfile).

Пример из IDA:

```cpp
if ( IoStatusBlock )
{
  IoStatusBlock->Pointer = (PVOID) 259 ;
  v13 = IoStatusBlock[ 1 ].Information;
  v14 = IoStatusBlock;
  if ( v12 == 9 )
  {
    if ( (v13 & 1 ) != 0 )
      v14 = 0 i64;
    v15 = NtFsControlFile(
            Handle,
            (HANDLE)v13,
            0 i64,
            v14,
            IoStatusBlock,
            FsControlCode,
            InputBuffer,
            InputBufferLength,
            OutputBuffer,
            OutputBufferLength);
}
else
{
  if ( (v13 & 1 ) != 0 )
    v14 = 0 i64;
  v15 = NtDeviceIoControlFile(
            Handle,
            (HANDLE)v13,
            0 i64,
            v14,
            IoStatusBlock,
            FsControlCode,
            InputBuffer,
            InputBufferLength,
            OutputBuffer,
            OutputBufferLength);
}
```


Таким образом вы можете найти много интересного во внутренней работе API. Adrien

Chevalier из amossys.fr провел исследование данной темы, которое вы можете прочитать [тут](https://blog.amossys.fr/filter-communication-ports.html).

Спасибо за внимание. Ждите новых статей.
