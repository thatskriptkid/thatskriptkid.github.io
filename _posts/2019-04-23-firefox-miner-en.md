---
layout: post
title: Analysis of trojan
tags: [windows, malware analysis]
category: [en]
---

One day I asked to analyze suspicious file named `FirefoxUpdate.exe`. It has hash value:

`MD5: 342E134B3DE901EE6A915909AECDAC4A`

EXE is not packed, written in C#. Two AV on Virustotal define it as `Win32.Trojan.WisdomEyes` and `UDS: DangerousObject.Multi.Generic`.

![Virustotal](/assets/images/ru/FirefoxUpdate/imgs/virustotal.png)

By examining pdb data inside file we can conclude that the user name on the developer's computer is Achraf

![pdb_path](/assets/images/ru/FirefoxUpdate/imgs/PDB_path.png)

  After decompiling the file, we can see that at startup, the page at `http://164.132.197.47/security-updates/available.php` is downloaded.

![AvailableChecking](/assets/images/ru/FirefoxUpdate/imgs/screenshot-from-2018-05-19-22-.png)

If the page contains the string _"available"_, then malware can start to work. Otherwise, a window is displayed with the text _"Unable to update, please try later"_

![UnableToUpdate](/assets/images/ru/FirefoxUpdate/imgs/Unable_to_update.png)

Next, exe creates a folder `C:/Program Files/ChromeUpdates/`. Depending whether the OS is 32- or 64-bit, this number is added to the address `http://164.132.197.47/security-updates/`. Three files are downloaded from resulting link: update.exe, config.json, ChromePassBackup.exe. Files are saved in the folder `C:/Program Files/ChromeUpdates/`.

![ChromeUpdateFolder](/assets/images/ru/FirefoxUpdate/imgs/ChromeUpdatesFolder.png)

Contents of the config.json file:

![Config json](/assets/images/ru/FirefoxUpdate/imgs/Config-json.png)

URLs tell us that this is Monero miner's config.

![MoneroSite](/assets/images/ru/FirefoxUpdate/imgs/Monero_site.png)

At the end of the download, a new cmd.exe process is created with a hidden window with the following arguments - `/C schtasks /create /tn SecurityUpdates /tr "C:\Program Files\ChromeUpdates\update.exe" /sc onstart /RU SYSTEM`.
From the command line, schtasks.exe is launched. This program creates (`/create`) a scheduled task with the name (`/tn`) SecurityUpdate, which will launch (`/tr`) the file `C:\Program Files\ChromeUpdates\update.exe`, at the start systems (`/sc onstart`) using the user SYSTEM (`/RU SYSTEM`).

![schtask](/assets/images/ru/FirefoxUpdate/imgs/schtasks.png)

Next, a string is generated that is 12 characters long, consisting of random characters picked from string `abcdefghijklmnopqrstuvwxyz0123456789`.

This string is substituted for NULL in `worker-id` field in the file `C:\Program Files\ChromeUpdates\config.json`. Looks like a random token for mining client.

![WorkerIdUrl](/assets/images/ru/FirefoxUpdate/imgs/Worker-id-url.png)

Next, a new cmd.exe process is created with the arguments `/C netsh interface ipv4 add dnsservers
NAME_OF_NONLOCAL_NETWORK_INTERFACE address = 1.1.1.1 index = 1.` This adds 1.1.1.1 on top of the dns list to all available network interfaces, except for Loopback.

![netsh](/assets/images/ru/FirefoxUpdate/imgs/netsh_dns.png)

At the end of the process, the `Update complete` window appears.

![UpdateComplete](/assets/images/ru/FirefoxUpdate/imgs/Update_Complite.png)

## ChromePassBackup.exe

The exe downloaded in the previous step is a utility for viewing passwords saved in Google Chrome [nirsoft](https://www.nirsoft.net/utils/chromepass.html).

## Update.exe

This file was scheduled. Virustotal defines the file as a miner. Miner repository: https://github.com/xmrig/xmrig/releases

![VirusTotalUpdate](/assets/images/ru/FirefoxUpdate/imgs/Update_EXE_Virustotal.png)

## System Restore & IOC

1) Delete the folder `C:/Program Files/ChromeUpdates/`

2) Block ip `hxxp://164(.)132(.)197(.)47/` in the firewall

3) Delete the scheduled SecurityUpdate task with the command:
`schtasks /Delete /TN SecurityUpdate`

4) Change the DNS server to the desired, instead of 1.1.1.1