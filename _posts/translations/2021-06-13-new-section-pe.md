---
layout: post
title: Добавление новой секции с кодом в PE файл
category: [translations]
tag: [windows, malware]
---

Всем привет! Прошло некоторое время с тех пор, как я постил что-то интересное, и я чувствую, что настал момент еще раз внести свой вклад в этот форум.

Я видел некоторых людей здесь, и на некоторых других форумах, которые делают вид, что понимают формат PE файла и понимают, как его модифицировать.
Большое количество кода на форуме - бесполезно, содержит баги или делает зараженный PE файл нестабильным или невалидным!

Данный код является введением в тему модификации PE.

Программа, которую я покажу вам, читает файл, проверяет, является ли он валидным по PE формату, и вставляет новую секцию. С целью показать именно заражение файла, я вставляю простую строку в только что созданную секцию, просто чтобы показать на наглядном примере!

Ниже код, который вы можете улучшать и делать все что захотите!
НАСЛАЖДАЙТЕСЬ!!!

```cpp
#include <stdio.h>
#include <windows.h>
 
DWORD align(DWORD size, DWORD align, DWORD addr){
    if (!(size % align))
        return addr + size;
    return addr + (size / align + 1) * align;
}
 
bool AddSection(char *filepath, char *sectionName, DWORD sizeOfSection){
    HANDLE file = CreateFile(filepath, GENERIC_READ | GENERIC_WRITE, 0, NULL, OPEN_EXISTING, FILE_ATTRIBUTE_NORMAL, NULL);
    if (file == INVALID_HANDLE_VALUE) 
        return false;
    DWORD fileSize = GetFileSize(file, NULL);
    // теперь мы знаем, какого размера память мы должны выделить для буфера
    BYTE *pByte = new BYTE[fileSize];
    DWORD dw;
    // давайте прочитаем весь файл, чтобы в дальнейшем использовать эту информацию о PE
    ReadFile(file, pByte, fileSize, &dw, NULL);
 
    PIMAGE_DOS_HEADER dos = (PIMAGE_DOS_HEADER)pByte;
    if (dos->e_magic != IMAGE_DOS_SIGNATURE)
        return false; //invalid PE
    PIMAGE_FILE_HEADER FH = (PIMAGE_FILE_HEADER)(pByte + dos->e_lfanew + sizeof(DWORD));
    PIMAGE_OPTIONAL_HEADER OH = (PIMAGE_OPTIONAL_HEADER)(pByte + dos->e_lfanew + sizeof(DWORD)+sizeof(IMAGE_FILE_HEADER));
    PIMAGE_SECTION_HEADER SH = (PIMAGE_SECTION_HEADER)(pByte + dos->e_lfanew + sizeof(IMAGE_NT_HEADERS));
 
    ZeroMemory(&SH[FH->NumberOfSections], sizeof(IMAGE_SECTION_HEADER));
    CopyMemory(&SH[FH->NumberOfSections].Name, sectionName, 8); 
    // Мы используем 8 байт для имени секции, потому что это максимальный разрешенный размер
 
    // вставляем всю необходимую информацию о нашей новой PE секции
    SH[FH->NumberOfSections].Misc.VirtualSize = align(sizeOfSection, OH->SectionAlignment, 0);
    SH[FH->NumberOfSections].VirtualAddress = align(SH[FH->NumberOfSections - 1].Misc.VirtualSize, OH->SectionAlignment, SH[FH->NumberOfSections - 1].VirtualAddress);
    SH[FH->NumberOfSections].SizeOfRawData = align(sizeOfSection, OH->FileAlignment, 0);
    SH[FH->NumberOfSections].PointerToRawData = align(SH[FH->NumberOfSections - 1].SizeOfRawData, OH->FileAlignment, SH[FH->NumberOfSections - 1].PointerToRawData);
    SH[FH->NumberOfSections].Characteristics = 0xE00000E0;
    /*
        0xE00000E0 = IMAGE_SCN_MEM_WRITE |
                     IMAGE_SCN_CNT_CODE  |
                     IMAGE_SCN_CNT_UNINITIALIZED_DATA  |
                     IMAGE_SCN_MEM_EXECUTE |
                     IMAGE_SCN_CNT_INITIALIZED_DATA |
                     IMAGE_SCN_MEM_READ 
    */
    SetFilePointer(file, SH[FH->NumberOfSections].PointerToRawData + SH[FH->NumberOfSections].SizeOfRawData, NULL, FILE_BEGIN);
    // перемещаем указатель позиции в файле на значение адреса последней секции + размер файла
    SetEndOfFile(file);
    // теперь изменим размер образа, для соответствия нашим модификациям
    // после добавления новой секции размер увеличился
    OH->SizeOfImage = SH[FH->NumberOfSections].VirtualAddress + SH[FH->NumberOfSections].Misc.VirtualSize;
    // так как мы добавили новую секцию, их количество увеличилось
    FH->NumberOfSections += 1;
    SetFilePointer(file, 0, NULL, FILE_BEGIN);
    // и в завершении, мы применяем все изменения
    WriteFile(file, pByte, fileSize, &dw, NULL);
    CloseHandle(file);
    return true;
}
 
bool AddCode(char *filepath){
    HANDLE file = CreateFile(filepath, GENERIC_READ | GENERIC_WRITE, 0, NULL, OPEN_EXISTING, FILE_ATTRIBUTE_NORMAL, NULL);
    if (file == INVALID_HANDLE_VALUE) 
        return false;
    DWORD filesize = GetFileSize(file, NULL);
    BYTE *pByte = new BYTE[filesize];
    DWORD dw;
    ReadFile(file, pByte, filesize, &dw, NULL);
    PIMAGE_DOS_HEADER dos = (PIMAGE_DOS_HEADER)pByte;
    PIMAGE_NT_HEADERS nt = (PIMAGE_NT_HEADERS)(pByte + dos->e_lfanew);
 
    // новая секция становится последней
    // мы должны получить адрес последней секции, чтобы вставить наши секретные данные
    PIMAGE_SECTION_HEADER first = IMAGE_FIRST_SECTION(nt);
    PIMAGE_SECTION_HEADER last = first + (nt->FileHeader.NumberOfSections - 1);
 
    SetFilePointer(file, last->PointerToRawData, NULL, FILE_BEGIN);
    char *str = "ATHENIAN WAS HERE";
    WriteFile(file, str, strlen(str), &dw, 0);
    CloseHandle(file);
    return TRUE;
}
 
void main()
{
    if (AddSection("C:\\Users\\M\\Desktop\\Athena.exe", ".ATH", 400)){
        printf("Section added!\n");
        // Вставляем данные в последнюю секцию
        if (AddCode("C:\\Users\\M\\Desktop\\Athena.exe")){
            printf("Code written!\n");
        }
        else
            printf("Error writting code!\n");
    }
    else
        printf("Error adding section!\n");
}
```