---
layout: post
title: A new approach to infecting Android applications
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

**Concept by Erbol and Thatskriptkid**

**Illustration by @alphin.fault on [Instagram](https://www.instagram.com/alphin.fault)**

**Article and proof-of-concept code by Thatskriptkid**

**[Proof-of-concept repository](https://github.com/thatskriptkid/apk-infector-Archinome-PoC)**

This article is intended for readers familiar with the conventional method of infecting Android applications by patching smali code who want to explore a different, more efficient approach. If you are unfamiliar with the conventional technique, read the "Creating payload" section of my article [How to steal a digital signature with a Man-in-the-Disk attack]({% post_url 2019-07-17-steal-ds-en %}). We developed the technique described here independently and could not find any prior public description of it.

Our technique:

1. Does not rely on Android vulnerabilities.
2. Is not intended for cracking applications, removing ads, or bypassing licenses.
3. Adds code without changing the target application's behavior or appearance.

## Disadvantages of the current approach

The conventional and widely used approach is to disassemble an application into smali code and patch it. [smali/baksmali](https://github.com/JesusFreke/smali) provides the underlying tooling for well-known APK infectors such as:

1. [backdoor-apk](https://github.com/dana-at-cp/backdoor-apk).
2. [TheFatRat](https://github.com/Screetsec/TheFatRat).
3. [apkwash](https://github.com/jbreed/apkwash).
4. [kwetza](https://github.com/sensepost/kwetza).

Malware also uses smali/baksmali-based patching. The diagram below shows the workflow of the [Android.InfectionAds.1](https://vms.drweb.com/virus/?i=17771929&lng=en) trojan:

![](/assets/images/infecting_android/img_for_paper/android_infection_ads_01_en.png)

Disassembling and patching modifies the original `classesN.dex` file, which creates two problems:

1. Additional code may exceed the [65,536-method limit](https://developer.android.com/studio/build/multidex#about) of [a single DEX file](https://github.com/JesusFreke/smali/issues/629).
2. The application may verify the integrity of its DEX files.

DEX disassembly is complex, requires continual tool maintenance, and is [highly dependent on the Android version](https://github.com/JesusFreke/smali/issues/595).

Almost all available infection and modification tools are written in Java or depend on the JVM. This limits where they can run and excludes routers, many embedded systems, and other environments without a JVM.

## Description of a new approach

Android has several application startup states. A cold start occurs when the system launches an application whose process is not already running.

![](/assets/images/infecting_android/img_for_paper/cold_launch.png)

Application initialization begins with the creation of an `Application` object. Many Android applications define a custom `Application` class that extends `android.app.Application`. For example:

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

The `test.pkg.TestApp` class must be registered in `AndroidManifest.xml`:

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

The startup sequence for this application is:

![](/assets/images/infecting_android/img_for_paper/Usual_app_launch.png)

This sequence suggests two basic requirements for the infection technique:

1. Execute the injected code when the application starts.
2. Preserve every step in the original application's startup sequence.
   
We inject the code during the cold-start sequence, between creation of the `Application` object and execution of its constructor. We create a new `Application` subclass, add it to the APK, and register it in `AndroidManifest.xml` in place of the original class. To preserve the startup chain, the injected class inherits from `test.pkg.TestApp`.

The injected `Application` class:

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

The resulting startup sequence is shown below, with modifications marked in red:

![](/assets/images/infecting_android/img_for_paper/Infected_app_launch.png)

The technique makes two changes:

1. Add `my.malicious.InjectedApp` to the original APK.
2. Replace `test.pkg.TestApp` with `my.malicious.InjectedApp` in `AndroidManifest.xml`.

## The benefits of the new approach

The required changes can be made:

1. Without decoding or re-encoding `AndroidManifest.xml`.
2. Without disassembling or reassembling DEX files.
3. Without modifying the original DEX files.

These properties make the technique applicable to a wide range of applications. Adding a class and patching the manifest is much faster than disassembling a DEX file. Because the injected class runs at the start of application initialization, its code executes immediately. With a few exceptions described below, the technique is independent of CPU architecture and Android version.

The demonstration PoC is written in Go and can be extended into a full-featured tool. It compiles to a single binary with no runtime dependencies, and Go's cross-compilation support makes it possible to build it for many operating systems and architectures.

We tested APKs modified by the PoC on:

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

We successfully tested the technique on many applications; their names are omitted for obvious reasons. This included applications that smali/baksmali could not disassemble and that existing infection tools therefore could not modify.

## Identifying necessary modifications in AndroidManifest.xml and patching

One required modification is to replace a string in `AndroidManifest.xml`. The string can be patched without decoding and re-encoding the manifest.

APKs store the manifest in a binary XML format. To make the format easier to inspect, I created a [Kaitai Struct description](https://github.com/thatskriptkid/Kaitai-Struct-Android-Manifest-binary-XML) that can also serve as documentation.

The structure of `AndroidManifest.xml`, with sizes in bytes shown in parentheses:

![](/assets/images/infecting_android/img_for_paper/Manifest_draw.png)

To identify which fields the patch affects, we built two applications that differed only in the length of their `Application` class names. We then unpacked the APKs and compared their binary manifests.

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

The fully qualified class name in the second manifest is nine characters longer. We opened both files in [HexCmp](https://www.fairdell.com/hexcmp/) and compared them.

The following table explains the differences between the manifests:

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
    <td class="tg-0pky">The original manifest has 0x2 padding bytes, which the modified version does not require.<br>Strings in the binary manifest use UTF-16, so each added character occupies 0x2 bytes.<br>Adding nine characters increases the string by 0x12 bytes; removing the 0x2 padding bytes produces a net difference of 0x10 bytes.</td>
  </tr>
  <tr>
    <td class="tg-0pky">header.string_table_len</td>
    <td class="tg-0pky">0xC</td>
    <td class="tg-0pky">Length of the string array</td>
    <td class="tg-0pky">0x10</td>
    <td class="tg-0pky">The modified value belongs to the string array. The 0x10-byte difference has the same cause as the change to header.file_len.</td>
  </tr>
  <tr>
    <td class="tg-0pky">string_offset_table.offset</td>
    <td class="tg-0pky">0x7C</td>
    <td class="tg-0pky">Offset of the string after the modified string</td>
    <td class="tg-0pky">0x12</td>
    <td class="tg-0pky">string_offset_table stores offsets into the manifest's string array. Because the modified string is 0x12 bytes longer, the following string moves forward by 0x12 bytes.<br>Padding does not affect this value because it appears before the string array.</td>
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
    <td class="tg-0pky">The number of characters added to the string</td>
  </tr>
</tbody>
</table>

![](/assets/images/infecting_android/img_for_paper/Manifest_diff_2.png)

In the manifest structure shown earlier, `padding` follows `strings` to align `resource_header`. In the original manifest, the final `uses-sdk` string ends at offset `0x322` (orange), so two padding bytes (green) are inserted before `resource_header`.

![](/assets/images/infecting_android/img_for_paper/Manifest_alignment_original.png)

In the modified manifest, `string_table` ends at offset `0x334` (orange) and is immediately followed by `resource_header` (red), so no padding is required.

![](/assets/images/infecting_android/img_for_paper/Manifest_alignment_longer.png)

The diagram below marks the fields that must be patched to replace the original `Application` class name with the injected one:

![](/assets/images/infecting_android/img_for_paper/Manifest_draw_modified.png)

The proof-of-concept code implements these changes in `manifest.Patch()`.

## Creating files to be injected in the target application

The second required modification is to add a class that contains the injected code. To preserve the original startup chain, the new `Application` class must inherit from the target's original `Application` class. Because that class name is unknown when the injection files are prepared, the new class initially uses the placeholder `z.z.z`.

The initial state of the target application and the DEX to be injected:

![](/assets/images/infecting_android/img_for_paper/DEX_initial_state.png)

After extracting the original `Application` class name from the manifest, the PoC replaces the placeholder:

![](/assets/images/infecting_android/img_for_paper/DEX_state_after_patching.png)

The process ends by adding the new DEX to the target application:

![](/assets/images/infecting_android/img_for_paper/DEX_state_after_injection.png)

The payload classes are stored in a separate DEX so they can vary independently and so that patching the placeholder remains simple:

![](/assets/images/infecting_android/img_for_paper/DEX_split.png)

Class names in a DEX file are sorted alphabetically, while the target application's class name may begin with any letter. We chose `z.z.z` as the placeholder to keep its position in the string table predictable after patching.

![](/assets/images/infecting_android/img_for_paper/DEX_strings.png)

We created an Android Studio project containing three classes to prepare the injection files.

The first class is `InjectedApp`, with the following fully qualified name:

`aaaaaaaa.aaaaaaaaaaaaaaaaaaaa.aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa.aaaaaaaaaaaaaaaaaaaaaa.InjectedApp`

The name must satisfy two requirements:

1. It must be longer than the `Application` class name in any target application.

2. It must sort before any possible target `Application` class name.

The `InjectedApp` class executes in place of the target application's original `Application` class:

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

Its purpose is to invoke code stored in another DEX:

```java
        payload p = new payload();
        p.executePayload();
```

The `payload` class contains the code to be executed:

```java
package aaaaaaaaaaaa;

import android.util.Log;

public class payload {

    public void executePayload() {
            Log.i("HELL", "Hello, I'm a malicious payload");
    }
}
```

Its fully qualified name must satisfy one requirement:

1. It must sort before any possible target `Application` class name.

To supply arbitrary code, create a DEX file that meets these conditions:

1. It contains a class named:

`aaaaaaaaaaaaaa.payload`

2. The class implements the method:

`public void executePayload()`

The third class is the `z.z.z` placeholder. Its fully qualified name will be replaced with the target application's `Application` class name.

```java
package z.z;

import android.app.Application;

public class z extends Application {
}
```

The placeholder must satisfy two requirements:

1. Its fully qualified name must sort after the names of `InjectedApp` and `payload`.

2. Its fully qualified name must be shorter than any target `Application` class name.

Following this design, we compile `InjectedApp` and `payload` into separate DEX files. Build the APK with **Android Studio > Generate Signed Bundle/APK > release**. The compiled `.class` files appear in `app\build\intermediates\javac\release\classes`.

Compile the `.class` files into DEX files with [d8](https://developer.android.com/studio/command-line/d8):

`d8 --release --min-api 16 --no-desugaring InjectedApp.class --output .`

`d8 --release --min-api 16 --no-desugaring payload.class --output .`

The resulting DEX files can then be added to the target application.

## Identifying the necessary modifications in DEX and patching

Replacing `z.z.z` with the target application's fully qualified `Application` class name changes the DEX structure. To identify every affected field, we created two Android Studio applications whose class names had different lengths.

In the first application, `InjectedApp` inherits from `z.z.z`:

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

In the second application, `InjectedApp` inherits from `z.z.zzzzzzzzzzzzzzzzzzzzzzzz`:

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

The second class name is 15 characters longer. We compiled each class into a separate DEX file:

`d8 --release --min-api 16 --no-desugaring InjectedApp.class --output .`

We then opened the resulting DEX files in HexCmp:

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
    <td class="tg-0pky">The checksum must be recalculated after any change to the DEX file.</td>
  </tr>
  <tr>
    <td class="tg-0pky">header_item.signature</td>
    <td class="tg-0pky">0xC</td>
    <td class="tg-0pky">Hash</td>
    <td class="tg-0pky">full</td>
    <td class="tg-0pky">The signature hash must be recalculated after any change to the DEX file.</td>
  </tr>
  <tr>
    <td class="tg-0pky">header_item.file_size</td>
    <td class="tg-0pky">0x20</td>
    <td class="tg-0pky">File size</td>
    <td class="tg-0pky">0x10</td>
    <td class="tg-0pky">The string grows by 0xF bytes and requires one additional padding byte.</td>
  </tr>
  <tr>
    <td class="tg-0pky">header_item.map_off</td>
    <td class="tg-0pky">0x34</td>
    <td class="tg-0pky">map offset</td>
    <td class="tg-0pky">0x10</td>
    <td class="tg-0pky">The map follows the string array, so its offset increases by the string growth plus padding.</td>
  </tr>
  <tr>
    <td class="tg-0pky">header_item.data_size</td>
    <td class="tg-0pky">0x68</td>
    <td class="tg-0pky">data section size</td>
    <td class="tg-0pky">0x10</td>
    <td class="tg-0pky">The data section follows the string array, so its size increases by the string growth plus padding.</td>
  </tr>
  <tr>
    <td class="tg-0pky">map.class_def_item.class_data_off</td>
    <td class="tg-0pky">0xE8</td>
    <td class="tg-0pky">offset to class data</td>
    <td class="tg-0pky">0xF</td>
    <td class="tg-0pky">This structure requires no alignment, so the offset increases only by the number of added bytes.</td>
  </tr>
  <tr>
    <td class="tg-0pky">map_list.debug_info_item</td>
    <td class="tg-0pky">0x114</td>
    <td class="tg-0pky">debug info offset</td>
    <td class="tg-0pky">Not important</td>
    <td class="tg-0pky">This field points to debugging metadata used for stack traces. The PoC can ignore it.</td>
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
    <td class="tg-0pky">These ASCII characters occupy one byte each in the DEX file's MUTF-8 encoding.</td>
  </tr>
</tbody>
</table>

![](/assets/images/infecting_android/img_for_paper/DEX_diff_2.png)

Additional changes appear at the end of the file:

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
    <td class="tg-0pky">class_data_item immediately follows the string array and requires no alignment.</td>
  </tr>
  <tr>
    <td class="tg-0pky">map.annotation_set_item.entries.annotation_off_item</td>
    <td class="tg-0pky">0x2A8</td>
    <td class="tg-0pky">offset to annotations</td>
    <td class="tg-0pky">0x10</td>
    <td class="tg-0pky">The offset includes alignment padding.</td>
  </tr>
  <tr>
    <td class="tg-0pky">map.map_list.offset</td>
    <td class="tg-0pky">0x2B4</td>
    <td class="tg-0pky">offset to map_list</td>
    <td class="tg-0pky">0x10</td>
    <td class="tg-0pky">The offset includes alignment padding.</td>
  </tr>
</tbody>
</table>

![](/assets/images/infecting_android/img_for_paper/DEX_diff_3.png)

The proof-of-concept code implements these changes in `mydex.Patch()`.

## Results

The PoC applies the required changes as follows:

1. Unpack the APK.
2. Parse `AndroidManifest.xml`.
3. Find the original `Application` class name.
4. Replace it with `aaaaaaaa.aaaaaaaaaaaaaaaaaaaa.aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa.aaaaaaaaaaaaaaaaaaaaaa.InjectedApp`.
5. Replace the `z.z.z` placeholder with the original `Application` class name.
6. Add two DEX files to the APK: one containing `InjectedApp`, and one containing the payload classes.
7. Package the files into a new APK.

## Limitations of the new approach

This technique does not work when all of the following conditions are true:

1. `minSdkVersion <= 20`.
2. The application does not depend on `androidx.multidex:multidex` or `com.android.support:multidex`.
3. The application runs on a version earlier than Android 5.0 (API level 21).

The technique also fails when the manifest sets `android:extractNativeLibs=false`. Installation then returns `INSTALL_FAILED_INVALID_APK: Failed to extract native libraries, res=-2`.

The first limitation applies to applications that originally contain only one DEX file. Android versions earlier than 5.0 use the Dalvik virtual machine, which supports only one DEX per APK by default. The multidex libraries listed above provide compatibility support. Android 5.0 and later use ART, which natively supports multiple DEX files and compiles them during installation. See the [official documentation](https://developer.android.com/studio/build/multidex) for details.

## Further PoC improvements

1. Add `InjectedApp` to `AndroidManifest.xml` when the target has no custom `Application` class.
2. Support adding new manifest elements.
3. Add APK signing.
4. Eliminate the remaining `AndroidManifest.xml` decoding step.
5. Handle `android:extractNativeLibs=false`.

## FAQ

Q: Why not use underscores in the fully qualified name of `InjectedApp`, making it almost certain to sort before the target application's `Application` class name?

A: Although technically possible, this causes the following error on Android 5:

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

Q: Why not inject an `Activity` and register it in place of the original launcher activity? The payload would run slightly later, but the delay would not matter.

A: This approach has two problems. First, some applications define many [`activity-alias`](https://developer.android.com/guide/topics/manifest/activity-alias-element) elements that refer to the main activity. Supporting them would require patching several manifest entries and make identifying the correct activity more difficult. Second, the main activity runs on the UI thread, which restricts what the injected code can do without disrupting the application.

Q: If an `Application` class cannot start background services, what useful code can it run?

A: This restriction was introduced in later Android versions and applies to applications generally, not specifically to the `Application` class. Foreground services remain available where ordinary background services are restricted.

Q: Why is the PoC not working?

A: Check that:

1. The original application works.
2. Every file path in the PoC is correct.
3. `apkinfector.log` contains no unexpected errors.
4. The patched `InjectedApp.dex` contains the original `Application` class name in the expected location.
5. The target defines and uses a custom `Application` class. The current PoC does not support targets without one.

If these checks do not help, try adjusting the `--min-api` parameter when compiling the classes. If the problem persists, open an issue on GitHub.

Q: Why inject code into the `Application` constructor instead of `onCreate()`?

A: Some applications declare a final `onCreate()` method in their `Application` class. Attempting to override it in the injected class causes Android to report an error:

```
06-28 07:27:59.770  2153  4539 I ActivityManager: Start proc 6787:xxxxxxxxx/u0a46 for activity xxxxxxxxx/.Main
06-28 07:27:59.813  6787  6787 I art     : Rejecting re-init on previously-failed class java.lang.Class<InjectedApp>:
 java.lang.LinkageError: Method void InjectedApp.onCreate() overrides final method in class LX/001; 
(declaration of 'InjectedApp' appears in /data/app/xxxxxxxxx-1/base.apk:classes2.dex)
```

The relevant check is visible in the [ART source code](https://android.googlesource.com/platform/art/+/refs/heads/master/runtime/class_linker.cc#6640):

```
if (super_method->IsFinal()) {
          ThrowLinkageError(klass.Get(), "Method %s overrides final method in class %s",
                            virtual_method->PrettyMethod().c_str(),
                            super_method->GetDeclaringClassDescriptor());
          return false;
        }
``` 

ART detects that the superclass method is final and throws a linkage error.

In Java, the compiler provides a no-argument constructor only when the class declares no constructors of its own. Because the injected class calls a no-argument superclass constructor, this might appear to be a compatibility problem. In practice, however, Android itself requires an `Application` class to have a no-argument constructor. A class without one already fails with the following error:

```
06-28 08:51:54.647  8343  8343 D AndroidRuntime: Shutting down VM
06-28 08:51:54.647  8343  8343 E AndroidRuntime: FATAL EXCEPTION: main
06-28 08:51:54.647  8343  8343 E AndroidRuntime: Process: xxxxxxxxx, PID: 8343
06-28 08:51:54.647  8343  8343 E AndroidRuntime: java.lang.RuntimeException: Unable to instantiate application xxxxxxxxx.YYYYYY: java.lang.InstantiationException: java.lang.Class<xxxxxxxxx.YYYYYY> has no zero argument constructor
06-28 08:51:54.647  8343  8343 E AndroidRuntime:        at android.app.LoadedApk.makeApplication(LoadedApk.java:802)
```
