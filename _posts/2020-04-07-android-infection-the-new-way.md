---
layout: post
title: Infecting android applications - The new way
tags: [android, malware]
category: [en]
---

![](/assets/images/infecting_android/silent_night.png)

- [Foreword](#foreword)
- [Disadvantages of the current approach](#disadvantages-of-the-current-approach)
- [Description of a new approach](#description-of-a-new-approach)
- [The benefits of the new approach](#the-benefits-of-the-new-approach)
- [Identifying necessary modifications in AndroidManifest.xml and patching](#identifying-necessary-modifications-in-androidmanifestxml-and-patching)
- [Creating files to be injected in the target application](#creating-files-to-be-injected-in-the-target-application)
- [Identifying the necessary modifications in DEX and patching](#identifying-the-necessary-modifications-in-dex-and-patching)
- [Results](#results)
- [Limitations of the new approach](#limitations-of-the-new-approach)
- [Further PoC improvements](#further-poc-improvements)
- [FAQ](#faq)

## Foreword

**Idea authors: Erbol & Thatskriptkid**

**Author of drawing:**

**Author of the article and proof-of-concept code: Thatskriptkid**

**[Proof-of-Concept Link](https://github.com/thatskriptkid/apk-infector-Archinome-PoC)**

Target audience of the article - people who have an idea of the current way of infecting android applications through smali code patching and want to learn about a new and more effective way. If you are not familiar with the current infection practice, read my article - [How to steal digital signature using Man-In-The-Disk]({% post_url en/2019-07-17-steal-ds-en %}), chapter - "Creating payload". The technique described here was completely invented by us; there is no description of such a method in the Internet. 

Our technique:
1. Does not use bugs or android vulnerabilities
2. Not intended for cracking applications (removing ads, licenses, etc.). 
3. Designed to add malicious code without any interference with the target application or its appearance.

## Disadvantages of the current approach

The way to inject malicious code by decoding the application to smali code and patching it is the only and widely practiced method to date. [smali/backsmali](https://github.com/JesusFreke/smali) is the only tool used for this. It is the basis for all known apk infectors, for example:

1. [backdoor-apk](https://github.com/dana-at-cp/backdoor-apk).
2. [TheFatRat](https://github.com/Screetsec/TheFatRat)
3. [apkwash](https://github.com/jbreed/apkwash)
4. [kwetza](https://github.com/sensepost/kwetza)

Malware also uses smali/backsmali and patching. The work algorithm of the Trojan [Android.InfectionAds.1](https://vms.drweb.com/virus/?i=17771929&lng=en):

![](/assets/images/infecting_android/img_for_paper/android_infection_ads_01_en.png)

Decoding and patching involves changing the original classesN.dex file. This leads to two problems:

1. Overstepping [the limit of 65536](https://developer.android.com/studio/build/multidex#about) methods [in one DEX file](https://github.com/JesusFreke/smali/issues/629) if there is too much malicious code.
2. The application can check the integrity of DEX files

DEX decoding/disassembling is a complex process that requires constant updating and [highly dependent on the android version](https://github.com/JesusFreke/smali/issues/595). 

Almost all available infection/modification tools are written in Java and/or depend on JVM - this greatly narrows the scope of use and makes it impossible to launch the infectors on routers, embedded systems, systems without JVM, etc.

## Description of a new approach

There are several types of starting applications in the android, one of which is called cold start. Cold start happens when application is started for the first time. 

![](/assets/images/infecting_android/img_for_paper/cold_launch.png)

The execution of an application starts with the creation of an Application object. Most android applications have their own Application class, which should extends the main class `android.app.Applciation`. An example of a class:

```java
package test.pkg;
import android.app.Application;
public class TestApp extends Application {

    public TestApp() {}

    @Override
    public void onCreate() {
        super.onCreate();
    }
}
```

The class `test.pkg.TestApp` should be registered in AndroidManifest.xml:

```xml
<?xml version="1.0" encoding="utf-8"?>
<manifest xmlns:android="http://schemas.android.com/apk/res/android"
    package="com.example">

    <application
        android:icon="@mipmap/ic_launcher"
        android:label="Test"
        android:roundIcon="@mipmap/ic_launcher_round"
        android:name="test.pkg.TestApp">
    </application>
</manifest>
```

The process of launching such an application:

![](/assets/images/infecting_android/img_for_paper/Usual_app_launch.png)

The basic requirements for our infection techniques have been defined:

1. Execution of malicious code, at application launch 
2. Saving all steps of the process of launching the original application
   
The injection of the malicious code took place at the stage of the *cold start*:*Application Object creation->Application Object Constructor*. A malicious Application class was created, injected in the APK and spelled out in *AndroidManifest.xml*, instead of the original one. To preserve the previous execution chain, it was inherited from `test.pkg.TestApp`.

Malicious Application class:

```java
package my.malicious;
import test.pkg;
public class InjectedApp extends TestApp {

    public InjectedApp() {
        super();
        executeMaliciousPayload();
    }
}
```

Modified AndroidManifest.xml:

```xml
<?xml version="1.0" encoding="utf-8"?>
<manifest xmlns:android="http://schemas.android.com/apk/res/android"
    package="com.example">

    <application
        android:icon="@mipmap/ic_launcher"
        android:label="Test"
        android:roundIcon="@mipmap/ic_launcher_round"
        android:name="my.malicious.InjectedApp">
    </application>
</manifest>
```

The process of launching malicious code inside an infected application (modifications are marked in red):

![](/assets/images/infecting_android/img_for_paper/Infected_app_launch.png)

Applied modifications:

1. The class `my.malicious.InjectedApp` was added to the original APK
2. In AndroidManifest.xml the line `test.pkg.TestApp` has been replaced with `my.malicious.InjectedApp`

## The benefits of the new approach

It is possible to apply necessary modifications to the APK:

1. Without AndroidManifest.xml decoding/encoding
2. Without DEX dissasembling/assembling
3. Without making changes to the original DEX files

These facts allow you to infect almost any existing application without restrictions. Adding your own class and modifying the manifest works much faster than decoding DEX. The malicious code injected by our technology starts immediately, as we are injected right at the beginning of the application launch process. The described infection technique doesn't depend on the architecture and version of the android (with a few exceptions). 

The PoC for demonstration was written in Go and is capable to be extended to a full featured tool. PoC is compiled into one binary file and does not use any runtime dependencies. Using Go allows using cross compilation to build an infector for almost any architecture and OS.

Testing of infected APK by PoC was on:

```
NOX player 6.6.0.8006-7.1.2700200616, Android 7.1.2 (API 25), ARMv7-32

NOX player 6.6.0.8006-7.1.2700200616, Android 5.1.1 (API 22), ARMv7-32

Android Studio Emulator, Android 5.0 (API 21), x86

Android Studio Emulator, Android 7.0 (API 24), x86

Android Studio Emulator, Android 9.0 (API 28), x86_64

Android Studio Emulator, Android 10.0 (API 29), x86

Android Studio Emulator, Android 10.0 (API 29), x86_64

Android Studio Emulator, Android API 30, x86

Xiaomi Mi A1
```

We managed to successfully infect a huge number of applications (for obvious reasons, the names are hidden). We managed to successfully infect applications that cannot be decoded using smali/backsmali, and therefore by existing tool.

## Identifying necessary modifications in AndroidManifest.xml and patching

One of the modifications required for the infection is to replace the string in AndroidManifest.xml. It is possible to patch the string without decoding/encoding the manifest. 

APKs contain the manifest in binary encoded form. The structure of the binary manifest is undocumented and represents a custom XML encoding algorithm from Google. For convenience, [was created](https://github.com/thatskriptkid/Kaitai-Struct-Android-Manifest-binary-XML) a description in [Kaitai Struct](https://kaitai.io/) that can be used as documentation.

AndroidManifest.xml structure (in brackets - size in bytes):

![](/assets/images/infecting_android/img_for_paper/Manifest_draw.png)

Two applications with different class names were developed to detect changes in the manifest, after patching the original Application class name to a malicious one. The applications were built into an APK and unpacked to produce binary manifests.

An example of the original manifest:

```xml
<?xml version="1.0" encoding="utf-8"?>
<manifest xmlns:android="http://schemas.android.com/apk/res/android"
    package="com.qoogle.service.outbound.thread.safe.eng.packages.packas.pack.level.random">

    <application
        android:icon="@mipmap/ic_launcher"
        android:label="MinDEX"
        android:roundIcon="@mipmap/ic_launcher_round"
        android:name="test.pkg.TestApp">
    </application>

</manifest>
```

An example of a patched manifest:

```xml
<?xml version="1.0" encoding="utf-8"?>
<manifest xmlns:android="http://schemas.android.com/apk/res/android"
    package="com.qoogle.service.outbound.thread.safe.eng.packages.packas.pack.level.random">

    <application
        android:icon="@mipmap/ic_launcher"
        android:label="MinDEX"
        android:roundIcon="@mipmap/ic_launcher_round"
        android:name="test.pkg.TestAppAAAAAAAAA">
    </application>

</manifest>
```

The length of the fully-qualified class name has increased by 9 characters. Both files were opened in [HexCmp](https://www.fairdell.com/hexcmp/), to get the diff. 

Changes to the manifest and explanation of reasons:

<style type="text/css">
.tg  {border-collapse:collapse;border-spacing:0;}
.tg td{border-color:black;border-style:solid;border-width:1px;font-family:Arial, sans-serif;font-size:14px;
  overflow:hidden;padding:10px 5px;word-break:normal;}
.tg th{border-color:black;border-style:solid;border-width:1px;font-family:Arial, sans-serif;font-size:14px;
  font-weight:normal;overflow:hidden;padding:10px 5px;word-break:normal;}
.tg .tg-0pky{border-color:inherit;text-align:left;vertical-align:top}
</style>
<table class="tg">
<thead>
  <tr>
    <th class="tg-0pky">field</th>
    <th class="tg-0pky">offset</th>
    <th class="tg-0pky">description</th>
    <th class="tg-0pky">diff_count</th>
    <th class="tg-0pky">explanation</th>
  </tr>
</thead>
<tbody>
  <tr>
    <td class="tg-0pky">header.file_len</td>
    <td class="tg-0pky">0x4</td>
    <td class="tg-0pky">Total file length</td>
    <td class="tg-0pky">0x10</td>
    <td class="tg-0pky">In the original manifest there was 0x2 bytes of alignment, in the modified version they are not required. <br>Strings in the binary manifest are stored in UTF-16 format, i.e. one character takes 0x2 bytes.<br>In total, we increased the string by 9 characters (0x12 bytes) minus 0x2 alignment bytes, it equals to 0x10 byte difference</td>.
  </tr>
  <tr>
    <td class="tg-0pky">header.string_table_len</td>
    <td class="tg-0pky">0xC</td>
    <td class="tg-0pky">Length of array of strings</td>
    <td class="tg-0pky">0x10</td>
    <td class="tg-0pky">The string is in an array of strings. The explanation for the 0x10 byte difference is the same as for header.file_len.</td>
  </tr>
  <tr>
    <td class="tg-0pky">string_offset_table.offset</td>
    <td class="tg-0pky">0x7C</td>
    <td class="tg-0pky">Offset to the line following the modified</td>
    <td class="tg-0pky">0x12</td>
    <td class="tg-0pky">string_offset_table stores offset up to strings in an array of manifest strings. Since the length of the string has increased, <br>the line following it has been moved further by 0x12 bytes. Alignment is not taken into account here, as it <br> is located before the array of strings.</td>.
  </tr>
</tbody>
</table>

![](/assets/images/infecting_android/img_for_paper/Manifest_diff_1.png)

<style type="text/css">
.tg  {border-collapse:collapse;border-spacing:0;}
.tg td{border-color:black;border-style:solid;border-width:1px;font-family:Arial, sans-serif;font-size:14px;
  overflow:hidden;padding:10px 5px;word-break:normal;}
.tg th{border-color:black;border-style:solid;border-width:1px;font-family:Arial, sans-serif;font-size:14px;
  font-weight:normal;overflow:hidden;padding:10px 5px;word-break:normal;}
.tg .tg-0pky{border-color:inherit;text-align:left;vertical-align:top}
</style>
<table class="tg">
<thead>
  <tr>
    <th class="tg-0pky">field</th>
    <th class="tg-0pky">offset</th>
    <th class="tg-0pky">description</th>
    <th class="tg-0pky">diff_count</th>
    <th class="tg-0pky">explanation</th>
  </tr>
</thead>
<tbody>
  <tr>
    <td class="tg-0pky">strings.len</td>
    <td class="tg-0pky">0x2EA</td>
    <td class="tg-0pky">String length</td>
    <td class="tg-0pky">0x9</td>
    <td class="tg-0pky">The number of characters by which the string has increased</td>
  </tr>
</tbody>
</table>

![](/assets/images/infecting_android/img_for_paper/Manifest_diff_2.png)

In the structure of the manifest given at the beginning, after `strings` follows `padding` to align `resource_header`. In the original manifest, the last line of `uses-sdk` ends on the offset `0x322` (orange), which means that two bytes of alignment (green) for `resource_header` have been added.

![](/assets/images/infecting_android/img_for_paper/Manifest_alignment_original.png)

In the modified version, `string_table` ends in offset `0x334` (orange) and then immediately follows `resource_header` (red), which does not require alignment.

![](/assets/images/infecting_android/img_for_paper/Manifest_alignment_longer.png)

AndroidManifest.xml structure scheme, with an indication of fields to be patched, to replace the name of the original Applciation class with a malicious one (marked in red):

![](/assets/images/infecting_android/img_for_paper/Manifest_draw_modified.png)

The Proof-of-Concept code developed for the article implements these modifications in the `manifest.Patch()` method.

## Creating files to be injected in the target application

The second modification needed to infect is the injection of a class with malicious code. In order to save the original application startup chain, an Application class must be injected into the APK, the parent class of which must be the original Application class. At the stage of preparing the files to be injected, it is unknown. Therefore, when creating the class, it was necessary to use the name placeholder - `z.z.z`. 

The initial state of the application and the DEX to be injected:

![](/assets/images/infecting_android/img_for_paper/DEX_initial_state.png)

After receiving the original name of the Application class from the manifest, the placeholder was patched:

![](/assets/images/infecting_android/img_for_paper/DEX_state_after_patching.png)

The infection process ends with the addition of the malicious DEX to the target application:

![](/assets/images/infecting_android/img_for_paper/DEX_state_after_injection.png)

Since classes with malicious code can have different code, they were put into a separate DEX. This was also done to simplify the patching of the placeholder:

![](/assets/images/infecting_android/img_for_paper/DEX_split.png)

The class names in DEX are arranged alphabetically. The Application class name of the target application can start with any letter. For predictable string order, after the patching, the name of the placeholder was chosen to be `z.z.z`. 

![](/assets/images/infecting_android/img_for_paper/DEX_strings.png)

To prepare the files to be injected, a project was created in Android Studio, with three classes.

Class `InjectedApp`. Its full name:

`aaaaaaaa.aaaaaaaaaaaaaaaaaaaa.aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa.aaaaaaaaaaaaaaaaaaaaaa.InjectedApp`

This name must meet two rules:

1. It must be longer than any Application class name of any target application

2. It must be higher in alphabetical order of any Application Class name of any application

The `InjectedApp` class that will be executed instead of the Application class of the target application:

```java
package aaaaaaaa.aaaaaaaaaaaaaaaaaaaa.aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa.aaaaaaaaaaaaaaaaaaaaaa;
import aaaaaaaaaaaa.payload;
import z.z.z;

public class InjectedApp extends z {

    public InjectedApp() {
        super();
        payload p = new payload();
        p.executePayload();
    }
}
```

The main goal of the class is to start executing a malicious code that is in another DEX:

```java
        payload p = new payload();
        p.executePayload();
```

The `payload` class contains malicious code:

```java
package aaaaaaaaaaaa;

import android.util.Log;

public class payload {

    public void executePayload() {
            Log.i("HELL", "Hello, I'm a malicious payload");
    }
}
```

The full name of the class must satisfy the following rule:

1. It must be alphabetically higher than any Application class name of any application 

To inject arbitrary malicious code, you must create a DEX file that must comply with the conditions:

1. Contain a class with a name:

`aaaaaaaaaaaaaa.payload`

2. The class must contain the method

`public void executePayload()`

A placeholder class `z.z.z`, whose full name will be patched to the full name of the Applciation class of the target application.

```java
package z.z;

import android.app.Application;

public class z extends Application {
}
```

The class must comply with the conditions:

1. The full name of the class must be alphabetically lower than the full names of the classes `InjectedApp` and `payload`

2. The full name of the class must be shorter than any of the full names of the Application classes of any application

According to the developed injection scheme, the `InjectedApp` and `payload` classes were compiled into separate DEXs. For this purpose, Android Studio built the APK with `Android Studio->Generate Signed Bundle/APK->release`. The compiled .class files were created in the folder `app\build\intermediates\javac\release\classes`.

Compile .class files into DEX, using [d8](https://developer.android.com/studio/command-line/d8):

`d8 --release --min-api 16 --no-desugaring InjectedApp.class --output .`

`d8 --release --min-api 16 --no-desugaring payload.class --output .`

The resulting DEX should be added to the target application.

## Identifying the necessary modifications in DEX and patching

After patching the placeholder `z.z.z` to the full name of the Application class of the target application, the DEX structure will change. To detect modifications, two applications with class names of different lengths were created in Android Studio.

The `InjectedApp` class, inherited from `z.z.z`, in the first application:

```java
package aaaaaaaa.aaaaaaaaaaaaaaaaaaaa.aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa.aaaaaaaaaaaaaaaaaaaaaa;
import aaaaaaaaaaaa.payload;
import z.z.z;

public class InjectedApp extends z {

    public InjectedApp() {
        super();
        payload p = new payload();
        p.executePayload();
    }
}
```

Class `InjectedApp`, inherited from `z.z.zzzzzzzzzzzzzzzzzzzzzzzz` in the second application:

```java
package aaaaaaaa.aaaaaaaaaaaaaaaaaaaa.aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa.aaaaaaaaaaaaaaaaaaaaaa;
import aaaaaaaaaaaa.payload;
import z.z.z;

public class InjectedApp extends zzzzzzzzzzzzzzzz {

    public InjectedApp() {
        super();
        payload p = new payload();
        p.executePayload();
    }
}
```

The length of the class name increased by 15 characters. Classes were compiled separately into DEX:

`d8 --release --min-api 16 --no-desugaring InjectedApp.class --output .`

Let's open the resulting DEX in HexCMP:

*[Official documentation on the DEX structure](https://source.android.com/devices/tech/dalvik/dex-format)*

<style type="text/css">
.tg  {border-collapse:collapse;border-spacing:0;}
.tg td{border-color:black;border-style:solid;border-width:1px;font-family:Arial, sans-serif;font-size:14px;
  overflow:hidden;padding:10px 5px;word-break:normal;}
.tg th{border-color:black;border-style:solid;border-width:1px;font-family:Arial, sans-serif;font-size:14px;
  font-weight:normal;overflow:hidden;padding:10px 5px;word-break:normal;}
.tg .tg-0pky{border-color:inherit;text-align:left;vertical-align:top}
.tg .tg-0pky{text-align:left;vertical-align:top}
</style>
<table class="tg">
<thead>
  <tr>
    <th class="tg-0pky">field</th>
    <th class="tg-0pky">offset</th>
    <th class="tg-0pky">description</th>
    <th class="tg-0pky">diff_count</th>
    <th class="tg-0pky">explanation</th>
  </tr>
</thead>
<tbody>
  <tr>
    <td class="tg-0pky">header_item.checksum</td>
    <td class="tg-0pky">0x8</td>
    <td class="tg-0pky">Checksum</td>
    <td class="tg-0pky">full</td>
    <td class="tg-0pky">Any change in DEX, the checksum is recalculated.</td>
  </tr>
  <tr>
    <td class="tg-0pky">header_item.signature</td>
    <td class="tg-0pky">0xC</td>
    <td class="tg-0pky">Hash</td>
    <td class="tg-0pky">full</td>
    <td class="tg-0pky">Any change in the DEX hash is recalculated</td>
  </tr>
  <tr>
    <td class="tg-0pky">header_item.file_size</td>
    <td class="tg-0pky">0x20</td>
    <td class="tg-0pky">File size</td>
    <td class="tg-0pky">0x10</td>
    <td class="tg-0pky">String size increased by 0xF, plus 0x1 bytes of alignment.</td>
  </tr>
  <tr>
    <td class="tg-0pky">header_item.map_off</td>
    <td class="tg-0pky">0x34</td>
    <td class="tg-0pky">map offset</td>
    <td class="tg-0pky">0x10</td>
    <td class="tg-0pky">the map goes after an array of strings, so the offset was increased, taking into account the alignment</td>
  </tr>
  <tr>
    <td class="tg-0pky">header_item.data_size</td>
    <td class="tg-0pky">0x68</td>
    <td class="tg-0pky">data section size</td>
    <td class="tg-0pky">0x10</td>
    <td class="tg-0pky">The data section is located after an array of strings, so the offset was enlarged, taking into account the alignment</td>
  </tr>
  <tr>
    <td class="tg-0pky">map.class_def_item.class_data_off</td>
    <td class="tg-0pky">0xE8</td>
    <td class="tg-0pky">offset to class data</td>
    <td class="tg-0pky">0xF</td>
    <td class="tg-0pky">This structure does not require alignment, so the value increased by the number of added characters</td>
  </tr>
  <tr>
    <td class="tg-0pky">map_list.debug_info_item</td>
    <td class="tg-0pky">0x114</td>
    <td class="tg-0pky">debug info offset</td>
    <td class="tg-0pky">Not important</td>
    <td class="tg-0pky">The field stores the data needed for the correct output when it is crashed. The field can be ignored.</td>
  </tr>
</tbody>
</table>

![](/assets/images/infecting_android/img_for_paper/DEX_diff_1.png)

<style type="text/css">
.tg  {border-collapse:collapse;border-spacing:0;}
.tg td{border-color:black;border-style:solid;border-width:1px;font-family:Arial, sans-serif;font-size:14px;
  overflow:hidden;padding:10px 5px;word-break:normal;}
.tg th{border-color:black;border-style:solid;border-width:1px;font-family:Arial, sans-serif;font-size:14px;
  font-weight:normal;overflow:hidden;padding:10px 5px;word-break:normal;}
.tg .tg-0pky{border-color:inherit;text-align:left;vertical-align:top}
</style>
<table class="tg">
<thead>
  <tr>
    <th class="tg-0pky">field</th>
    <th class="tg-0pky">offset</th>
    <th class="tg-0pky">description</th>
    <th class="tg-0pky">diff_count</th>
    <th class="tg-0pky">explanation</th>
  </tr>
</thead>
<tbody>
  <tr>
    <td class="tg-0pky">string_data_item.utf16_size</td>
    <td class="tg-0pky">0x1B3</td>
    <td class="tg-0pky">string length</td>
    <td class="tg-0pky">0xF</td>
    <td class="tg-0pky">Strings in DEX are stored in MUTF-8 format, where one character takes 1 byte.</td>
  </tr>
</tbody>
</table>

![](/assets/images/infecting_android/img_for_paper/DEX_diff_2.png)

Changes at the end of the file:

<style type="text/css">
.tg  {border-collapse:collapse;border-spacing:0;}
.tg td{border-color:black;border-style:solid;border-width:1px;font-family:Arial, sans-serif;font-size:14px;
  overflow:hidden;padding:10px 5px;word-break:normal;}
.tg th{border-color:black;border-style:solid;border-width:1px;font-family:Arial, sans-serif;font-size:14px;
  font-weight:normal;overflow:hidden;padding:10px 5px;word-break:normal;}
.tg .tg-0pky{border-color:inherit;text-align:left;vertical-align:top}
.tg .tg-0pky{text-align:left;vertical-align:top}
</style>
<table class="tg">
<thead>
  <tr>
    <th class="tg-0pky">field</th>
    <th class="tg-0pky">offset</th>
    <th class="tg-0pky">description</th>
    <th class="tg-0pky">diff_count</th>
    <th class="tg-0pky">explanation</th>
  </tr>
</thead>
<tbody>
  <tr>
    <td class="tg-0pky">map.class_data_item.offset</td>
    <td class="tg-0pky">0x29C</td>
    <td class="tg-0pky">offset to class data</td>
    <td class="tg-0pky">0xF</td>
    <td class="tg-0pky">The structure class_data_item follows immediately after an array of strings and does not require alignment</td>
  </tr>
  <tr>
    <td class="tg-0pky">map.annotation_set_item.entries.annotation_off_item</td>
    <td class="tg-0pky">0x2A8</td>
    <td class="tg-0pky">offset to annotations</td>
    <td class="tg-0pky">0x10</td>
    <td class="tg-0pky">The alignment is taken into account</td>
  </tr>
  <tr>
    <td class="tg-0pky">map.map_list.offset</td>
    <td class="tg-0pky">0x2B4</td>
    <td class="tg-0pky">offset to map_list</td>
    <td class="tg-0pky">0x10</td>
    <td class="tg-0pky">The alignment is taken into account</td>
  </tr>
</tbody>
</table>

![](/assets/images/infecting_android/img_for_paper/DEX_diff_3.png)

The Proof-of-Concept code developed for the article implements these modifications in the `mydex.Patch()` method.

## Results

To apply the necessary modifications, we have developed PoC, which works according to the algorithm:

1. Unpacking APK files
2. Parsing AndroidManifest.xml
3. Finding the name of the Application class
4. Patching original Application class name with `aaaaaaaa.aaaaaaaaaaaaaaaaaaaa.aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa.aaaaaaaaaaaaaaaaaaaaaa.InjectedApp`
5. Patching of the placeholder `z.z.z` with the original name of the Application class
6. Adding two DEXs to APK (one with InjectedApp application class, another with malicious classes)
7. Packing all files in new APK

## Limitations of the new approach 

This technique will not work with applications that meet all conditions simultaneously:

1. minSdkVersion <= 20
2. Do not use in dependencies library `androidx.multidex:multidex` or `com.android.support:multidex`.
3. Runs on android versions lower than Android 5.0 (API level 21). 

Thus, it is assumed that the application has one DEX file. The restriction applies because the android versions before Android 5.0 (API level 21) use the Dalvik virtual machine to run the code. By default, Dalvik only accepts a single DEX file in the APK. To get around this limitation, you should use the above libraries. Android versions after Android 5.0 (API level 21), instead of Dalvik, use the ART system, which natively supports multiple DEX files in an application, because when you install an application, it will compile all DEXs into one .oat file. See [official documentation](https://developer.android.com/studio/build/multidex) for details.

## Further PoC improvements

1. If an application does not have its own Application class, you should add `InjectedApp` to AndroidManifest.xml
2. Adding your tags to AndroidManifest.xml
3. APK signing
4. Getting rid of AndroidManifest.xml decoding

## FAQ

Q: Why not use underscores in the full name of InjectedApp, so it is almost guaranteed to be alphabetically above any name in the Application class of the target application?

A: Technically it's possible, but there will be problems with Android 5 and there will be the following error:

```
D/AndroidRuntime( 3891): Calling main entry com.android.commands.pm.Pm
D/DefContainer( 3414): Copying /mnt/shared/App/20200629234847850.apk to base.apk
W/PackageManager( 1802): Failed parse during installPackageLI
W/PackageManager( 1802): android.content.pm.PackageParser$PackageParserException: /data/app/vmdl1642407162.tmp/base.apk (at Binary XML file line #48): Bad class name ________.__________._0000000000000000000000000000000000000000000000000000000000000000.InjectedApp in package XXXXXXXXXXXXXXXXXXXx
W/PackageManager( 1802):        at android.content.pm.PackageParser.parseBaseApk(PackageParser.java:885)
W/PackageManager( 1802):        at android.content.pm.PackageParser.parseClusterPackage(PackageParser.java:790)
W/PackageManager( 1802):        at android.content.pm.PackageParser.parsePackage(PackageParser.java:754)
W/PackageManager( 1802):        at com.android.server.pm.PackageManagerService.installPackageLI(PackageManagerService.java:10816)
W/PackageManager( 1802):        at com.android.server.pm.PackageManagerService.access$2300(PackageManagerService.java:236)
W/PackageManager( 1802):        at com.android.server.pm.PackageManagerService$6.run(PackageManagerService.java:8888)
W/PackageManager( 1802):        at android.os.Handler.handleCallback(Handler.java:739)
W/PackageManager( 1802):        at android.os.Handler.dispatchMessage(Handler.java:95)
W/PackageManager( 1802):        at android.os.Looper.loop(Looper.java:135)
W/PackageManager( 1802):        at android.os.HandlerThread.run(HandlerThread.java:61)
W/PackageManager( 1802):        at com.android.server.ServiceThread.run(ServiceThread.java:46)
```

Q: Why not inject Activity and write it in the manifest instead of the main one, because it also starts first? Yes, with this method, payload will run a little later, but it's not critical.

A: There are two problems in this approach. The first is that there are applications that use a lot of tags [activity-alias](https://developer.android.com/guide/topics/manifest/activity-alias-element) in the manifest  that refer to the name of the main activity. In this case we will have to patch not one line in the manifest, but several. It also makes it difficult to parse and find the name of the desired Activity. The second is that the main Activity runs in the main UI thread, which imposes some restrictions on the malicious code.

Q: But you can't use services in an Application class. What kind of malicious code can there be without services?

A: First of all, this restriction is introduced in the Android version starting with API 25. Secondly, this limitation applies to the android applications in general, not to the Application class specifically. Third, you can use services, but not ordinary services, but foreground. 

Q: Your PoC is not working

A: In this case, make sure that:
   1. The original application works
   2. All file paths in PoC are correct
   3. There's nothing unusual in apkinfector.log.
   4. The name of the original Application class in the patched InjectedApp.dex is really in its place. 
   5. The target application uses its Application class. Otherwise, PoC inoperability is predictable.

If nothing helped, try to play with the `-min-api` parameter when compiling classes.
If nothing worked, then create an issue on github.

Q: Why was the Application constructor selected for the infection and not the OnCreate() method? 

A: The point is that there are applications that have an Application class that has the OnCreate() method with the final modifier. If you put your Application with OnCreate(), the android will generate an error:

```
06-28 07:27:59.770  2153  4539 I ActivityManager: Start proc 6787:xxxxxxxxx/u0a46 for activity xxxxxxxxx/.Main
06-28 07:27:59.813  6787  6787 I art     : Rejecting re-init on previously-failed class java.lang.Class<InjectedApp>:
 java.lang.LinkageError: Method void InjectedApp.onCreate() overrides final method in class LX/001; 
(declaration of 'InjectedApp' appears in /data/app/xxxxxxxxx-1/base.apk:classes2.dex)
```

Reasons for the error [here](https://android.googlesource.com/platform/art/+/refs/heads/master/runtime/class_linker.cc#6640)

```
if (super_method->IsFinal()) {
          ThrowLinkageError(klass.Get(), "Method %s overrides final method in class %s",
                            virtual_method->PrettyMethod().c_str(),
                            super_method->GetDeclaringClassDescriptor());
          return false;
        }
``` 

The Android detects that the super method is final and gives out an error.

In Java, if you have not created any constructor, the compiler will create it for you (without parameters). If you have created a constructor with parameters, then the constructor without parameters is not automatically created. Since we call a constructor without parameters, you may think that there is a problem if the target application's app class contains a constructor with parameters. But it is not correct because Android requires a default constructor. Otherwise, you get this error.

```
06-28 08:51:54.647  8343  8343 D AndroidRuntime: Shutting down VM
06-28 08:51:54.647  8343  8343 E AndroidRuntime: FATAL EXCEPTION: main
06-28 08:51:54.647  8343  8343 E AndroidRuntime: Process: xxxxxxxxx, PID: 8343
06-28 08:51:54.647  8343  8343 E AndroidRuntime: java.lang.RuntimeException: Unable to instantiate application xxxxxxxxx.YYYYYY: java.lang.InstantiationException: java.lang.Class<xxxxxxxxx.YYYYYY> has no zero argument constructor
06-28 08:51:54.647  8343  8343 E AndroidRuntime:        at android.app.LoadedApk.makeApplication(LoadedApk.java:802)
```
