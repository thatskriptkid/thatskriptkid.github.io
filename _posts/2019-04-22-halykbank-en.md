---
layout: post
title: Analysis of phishing attack on HalykBank customers
tags: [windows, malware analysis]
category: [en]
---

> I presented this at [2600 Qazaqstan](https://2600.kz/) (December 7, 2018).
> You can download a sample of malware [here](/assets/images/ru/HalykBankPhishing/payload.zip), password: **infected**. Do it at your own risk!

The following e-mail with an attachment, in the form of an .iso file, was sent to the customers of [HalyBank](https://halykbank.kz/).

Title:

![](/assets/images/ru/HalykBankPhishing/1-header%20email.png)

E-mail body:

![](/assets/images/ru/HalykBankPhishing/2-body%20email.png)

If we download the attached file, we will see that it is an .iso file, and contains _Scan00987643.exe_.

![](/assets/images/ru/HalykBankPhishing/3-iso.png)

Why iso file? Perhaps the attackers tried to bypass the detection when running inside AV VM.

Virustotal result:

![](/assets/images/ru/HalykBankPhishing/4-VirusTotal.png)

Virustotal incorrectly determined type of malware and what its code name is, lately we will see why that happened. Let's take a closer look at the file:

![](/assets/images/ru/HalykBankPhishing/5-DIE.png)

Exe is written in C#, obfuscated using [.NET Reactor](https://www.eziriz.com/dotnet_reactor.htm). We can remove obfuscation using [de4dot](https://github.com/0xd4d/de4dot)

![](/assets/images/ru/HalykBankPhishing/6-de4dot%20description.png)

![](/assets/images/ru/HalykBankPhishing/6-de4dot%20console.png)

Good job! Open the cleaned file in hex editor trying to search for oddities and find such a pattern

![](/assets/images/ru/HalykBankPhishing/7-in%20hex%20editor.png)

Most like xored data. 

Nothing more interesting was found during the static analysis, let's start the debugging! A great utility will help us in this - [dnSpy](https://github.com/0xd4d/dnSpy)! It is a debugger and .NET decompiler. dnSpy allows us edit application's source code in runtime. Moreover, application can be written using the .NET Framework, .NET Core, and Unity

Having opened the exe in dnSpy, we see that it was not completely deobfuscated

![](/assets/images/ru/HalykBankPhishing/9-obfuscate.png)

After reading the source code near the entry point, we can see that the malware loads malicious payload

![](/assets/images/ru/HalykBankPhishing/10-loading%20payload.png)

`Assembly.Load()` accepts a byte array with a malicious paylod. DnSpy allows us to edit source code

![](/assets/images/ru/HalykBankPhishing/11-dnsPy%20edit.png)

Change the payload loading so we can save it to file

![](/assets/images/ru/HalykBankPhishing/12-editing%20function.png)

Hooray! We dumped the first payload!

![](/assets/images/ru/HalykBankPhishing/13-extracted%20first%20payload.png)

Payload is also written in C# and obfuscated (DIE did not show this) * (

![](/assets/images/ru/HalykBankPhishing/14-DIE%20on%20payload.png)

For now we have an ISO file contains an obfuscated EXE that loads another EXE, which is also obfuscated.

Use de4dot again

![](/assets/images/ru/HalykBankPhishing/16-de4dot%20second%20payload.png)

Load the resulting exe into dnSpy and read the source again. We see that all the strings in malware are encoded and decoded in runtime.

![](/assets/images/ru/HalykBankPhishing/17-encoded%20strings-1.png)

![](/assets/images/ru/HalykBankPhishing/17-encoded%20strings-2.png)

![](/assets/images/ru/HalykBankPhishing/17-encoded%20strings-3.png)

Resource section with Base64 encoded strings.

![](/assets/images/ru/HalykBankPhishing/18-strings%20in%20resources.png)

Payload copies itself to the `Users` folder under the name "null" and is added to autorun

![](/assets/images/ru/HalykBankPhishing/19-payload%20actions-1.png)

![](/assets/images/ru/HalykBankPhishing/19-payload%20actions-2.png)

From the resources dll is loaded. DLL takes decoded exe from resource as a parameter

![](/assets/images/ru/HalykBankPhishing/20-load%20DLL.png)

![](/assets/images/ru/HalykBankPhishing/20-load%20DLL-2.png)

This obfuscated exe has the strange name - **Reborn stub.**

![](/assets/images/ru/HalykBankPhishing/26-reborn%20stub.png)

Earlier, I heard about it and therefore I decided to google this name

![](/assets/images/ru/HalykBankPhishing/27-reborn%20stub%20on%20google.png)

It turned out that this payload is popular Hawkeye keylogger. It was decided not to analyze further manually, but to use the [any.run](https://any.run/), for further dynamic analysis

![](/assets/images/ru/HalykBankPhishing/28-any%20run.png)

RebornStub.exe work chain 

![](/assets/images/ru/HalykBankPhishing/29-any%20run%202.png)

This is Hawkeye Keylogger! If you remember, initially Virustotal did not define it that way.

![](/assets/images/ru/HalykBankPhishing/30-any%20run%203.png)

Strange outside connect

![](/assets/images/ru/HalykBankPhishing/31-any%20run%204.png)

Malware loads Mozilla dll in order to steal Firefox passwords!

![](/assets/images/ru/HalykBankPhishing/32-any%20run%205.png)

The general scheme

![](/assets/images/ru/HalykBankPhishing/33-any%20run%206.png)