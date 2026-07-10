---
layout: post
title: Analysis of a fake Firefox update trojan
tags: [windows, malware analysis]
category: [en]
---

I was asked to analyze a suspicious file named `FirefoxUpdate.exe`. Its hash is:

`MD5: 342E134B3DE901EE6A915909AECDAC4A`

The executable is not packed and was written in C#. Two antivirus engines on VirusTotal detect it as `Win32.Trojan.WisdomEyes` and `UDS:DangerousObject.Multi.Generic`.

![Virustotal](/assets/images/ru/FirefoxUpdate/imgs/virustotal.png)

The PDB path embedded in the file suggests that the username on the developer's computer was `Achraf`.

![pdb_path](/assets/images/ru/FirefoxUpdate/imgs/PDB_path.png)

Decompiling the file shows that it requests `http://164.132.197.47/security-updates/available.php` on startup.

![AvailableChecking](/assets/images/ru/FirefoxUpdate/imgs/screenshot-from-2018-05-19-22-.png)

If the response contains the string _"available"_, the malware continues. Otherwise, it displays a window with the message _"Unable to update, please try later"_.

![UnableToUpdate](/assets/images/ru/FirefoxUpdate/imgs/Unable_to_update.png)

Next, the executable creates `C:/Program Files/ChromeUpdates/`. Depending on whether the operating system is 32-bit or 64-bit, the malware appends the corresponding number to `http://164.132.197.47/security-updates/`. It then downloads three files from the resulting URL: `update.exe`, `config.json`, and `ChromePassBackup.exe`. All three are saved in `C:/Program Files/ChromeUpdates/`.

![ChromeUpdateFolder](/assets/images/ru/FirefoxUpdate/imgs/ChromeUpdatesFolder.png)

The contents of `config.json`:

![Config json](/assets/images/ru/FirefoxUpdate/imgs/Config-json.png)

The URLs indicate that this is a Monero miner configuration.

![MoneroSite](/assets/images/ru/FirefoxUpdate/imgs/Monero_site.png)

After the download completes, the malware starts a hidden `cmd.exe` process with the following arguments: `/C schtasks /create /tn SecurityUpdates /tr "C:\Program Files\ChromeUpdates\update.exe" /sc onstart /RU SYSTEM`.

This launches `schtasks.exe`, which creates (`/create`) a scheduled task named (`/tn`) `SecurityUpdates`. At system startup (`/sc onstart`), the task runs (`/tr`) `C:\Program Files\ChromeUpdates\update.exe` as `SYSTEM` (`/RU SYSTEM`).

![schtask](/assets/images/ru/FirefoxUpdate/imgs/schtasks.png)

Next, the malware generates a 12-character random string using characters from `abcdefghijklmnopqrstuvwxyz0123456789`.

It replaces `NULL` in the `worker-id` field of `C:\Program Files\ChromeUpdates\config.json`. This appears to be a random identifier for the mining client.

![WorkerIdUrl](/assets/images/ru/FirefoxUpdate/imgs/Worker-id-url.png)

The malware then creates another `cmd.exe` process with the arguments `/C netsh interface ipv4 add dnsservers
NAME_OF_NONLOCAL_NETWORK_INTERFACE address = 1.1.1.1 index = 1.` This adds `1.1.1.1` as the preferred DNS server for every available network interface except the loopback interface.

![netsh](/assets/images/ru/FirefoxUpdate/imgs/netsh_dns.png)

Finally, an `Update complete` window appears.

![UpdateComplete](/assets/images/ru/FirefoxUpdate/imgs/Update_Complite.png)

## ChromePassBackup.exe

The executable downloaded in the previous step is [NirSoft ChromePass](https://www.nirsoft.net/utils/chromepass.html), a utility for viewing passwords saved by Google Chrome.

## Update.exe

This is the file launched by the scheduled task. VirusTotal identifies it as a miner. Its source is available in the [XMRig repository](https://github.com/xmrig/xmrig/releases).

![VirusTotalUpdate](/assets/images/ru/FirefoxUpdate/imgs/Update_EXE_Virustotal.png)

## System recovery and IOCs

1. Delete the folder `C:/Program Files/ChromeUpdates/`.

2. Block the IP address `164.132.197.47` in the firewall.

3. Delete the scheduled `SecurityUpdates` task with the command:
`schtasks /Delete /TN SecurityUpdates`

4. Replace `1.1.1.1` with the intended DNS server.
