---
layout: post
title: Cracking "OWASP UnCrackable App for Android Level 1" and learning smali
tags: [android, malware]
category: [en]
---

This article is about solution of [OWASP android crackme](https://github.com/OWASP/owasp-mstg/tree/master/Crackmes). These crackmes are given as demo examples in the [OWASP Mobile Security Testing Guide](https://www.owasp.org/index.php/OWASP_Mobile_Security_Testing_Guide). I have not read this guide. I will try to solve crackmes based only on my knowledge about the android. 

## UnCrackable App for Android Level 1

The first level meets us with the description:

<pre>
This app holds a secret inside. Can you find it?
Objective: A secret string is hidden somewhere in this app. Find a way to extract it.
</pre>

Download apk, install on android emulator, run. We see this window.

![](/assets/images/ru/owasp-1/1.png)

In the background, we see a form that we should probably enter. But the form is not available, the application detected that it was running on the emulator (rutual) and closed. Let's first look at where this window comes from and try to remove it. To do that, we need to look at the apk source code and change it. There are two ways to get the source code of an application - decompiling it and decoding it into bytecode. Decompiling will give us java classes that are easier to read. But we are more likely to be unable to make changes to them and rebuild them into apk. So we will use apktool to get the source code as smali. It is harder to read, but when we change it, the application will be painlessly reassembled. We'll decode it.

Translated with www.DeepL.com/Translator (free version)