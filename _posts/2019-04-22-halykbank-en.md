---
layout: post
title: Analysis of a phishing attack against Halyk Bank customers
tags: [windows, malware analysis]
category: [en]
---

> I presented this at [2600 Qazaqstan](https://2600.kz/) (December 7, 2018).
> You can download the malware sample [here](/assets/images/ru/HalykBankPhishing/payload.zip). The password is **infected**. Use it at your own risk.

The following email, which included an ISO attachment, was sent to customers of [Halyk Bank](https://halykbank.kz/).

Title:

![](/assets/images/ru/HalykBankPhishing/1-header%20email.png)

Email body:

![](/assets/images/ru/HalykBankPhishing/2-body%20email.png)

The attached ISO image contains a file named _Scan00987643.exe_.

![](/assets/images/ru/HalykBankPhishing/3-iso.png)

Why use an ISO file? The attackers may have been trying to evade detection in antivirus sandboxes.

VirusTotal results:

![](/assets/images/ru/HalykBankPhishing/4-VirusTotal.png)

VirusTotal did not initially identify the malware correctly. We will see why later. First, let's take a closer look at the file:

![](/assets/images/ru/HalykBankPhishing/5-DIE.png)

The executable is written in C# and obfuscated with [.NET Reactor](https://www.eziriz.com/dotnet_reactor.htm). We can remove the obfuscation with [de4dot](https://github.com/0xd4d/de4dot).

![](/assets/images/ru/HalykBankPhishing/6-de4dot%20description.png)

![](/assets/images/ru/HalykBankPhishing/6-de4dot%20console.png)

With that done, we can open the cleaned file in a hex editor and look for anomalies. One pattern immediately stands out:

![](/assets/images/ru/HalykBankPhishing/7-in%20hex%20editor.png)

This is most likely XOR-encoded data.

Static analysis revealed nothing else of interest, so let's debug the sample with [dnSpy](https://github.com/0xd4d/dnSpy), a .NET debugger and decompiler. dnSpy can edit an application's code at runtime and supports applications built with .NET Framework, .NET Core, and Unity.

Opening the executable in dnSpy shows that it was not completely deobfuscated.

![](/assets/images/ru/HalykBankPhishing/9-obfuscate.png)

The decompiled code near the entry point shows that the malware loads another payload.

![](/assets/images/ru/HalykBankPhishing/10-loading%20payload.png)

The `Assembly.Load()` call receives the malicious payload as a byte array. Because dnSpy lets us edit the code, we can intercept it.

![](/assets/images/ru/HalykBankPhishing/11-dnsPy%20edit.png)

We modify the loading routine so that it saves the payload to a file.

![](/assets/images/ru/HalykBankPhishing/12-editing%20function.png)

Hooray! We dumped the first payload!

![](/assets/images/ru/HalykBankPhishing/13-extracted%20first%20payload.png)

The payload is also a C# executable and is obfuscated, although Detect It Easy did not identify the obfuscator.

![](/assets/images/ru/HalykBankPhishing/14-DIE%20on%20payload.png)

At this point, we have an ISO image containing an obfuscated executable that loads another obfuscated executable.

We use de4dot again.

![](/assets/images/ru/HalykBankPhishing/16-de4dot%20second%20payload.png)

We load the resulting executable into dnSpy and inspect the code again. All of the malware's strings are encoded and decoded at runtime.

![](/assets/images/ru/HalykBankPhishing/17-encoded%20strings-1.png)

![](/assets/images/ru/HalykBankPhishing/17-encoded%20strings-2.png)

![](/assets/images/ru/HalykBankPhishing/17-encoded%20strings-3.png)

The resources section contains Base64-encoded strings.

![](/assets/images/ru/HalykBankPhishing/18-strings%20in%20resources.png)

The payload copies itself to the `Users` directory under the name `null` and establishes persistence through an autorun entry.

![](/assets/images/ru/HalykBankPhishing/19-payload%20actions-1.png)

![](/assets/images/ru/HalykBankPhishing/19-payload%20actions-2.png)

A DLL is loaded from the resources and receives the decoded executable, also extracted from the resources, as an argument.

![](/assets/images/ru/HalykBankPhishing/20-load%20DLL.png)

![](/assets/images/ru/HalykBankPhishing/20-load%20DLL-2.png)

This obfuscated executable has an unusual name: **Reborn stub**.

![](/assets/images/ru/HalykBankPhishing/26-reborn%20stub.png)

I had heard the name before, so I searched for it.

![](/assets/images/ru/HalykBankPhishing/27-reborn%20stub%20on%20google.png)

The payload turned out to be the popular HawkEye keylogger. Instead of continuing the manual analysis, I used [ANY.RUN](https://any.run/) for further dynamic analysis.

![](/assets/images/ru/HalykBankPhishing/28-any%20run.png)

The `RebornStub.exe` execution chain:

![](/assets/images/ru/HalykBankPhishing/29-any%20run%202.png)

This confirms that the sample is the HawkEye keylogger, which VirusTotal did not initially identify correctly.

![](/assets/images/ru/HalykBankPhishing/30-any%20run%203.png)

The sample makes a suspicious outbound connection.

![](/assets/images/ru/HalykBankPhishing/31-any%20run%204.png)

The malware loads a Mozilla DLL to steal saved Firefox passwords.

![](/assets/images/ru/HalykBankPhishing/32-any%20run%205.png)

The overall execution flow:

![](/assets/images/ru/HalykBankPhishing/33-any%20run%206.png)
