---
layout: post
title: Calculation of approximate (minimum) timestamp for golang elf malware
tags: [linux, malware analysis]
category: [en]
---

## Intro

Timestamp (or compilation date) is one of the main characteristics of malware. It allows you to understand whether the sample is new, whether the sample has been used in other attacks and give an idea of when the attack was approximately prepared. But what if the malware sample has ELF format, which, as you know, does not save the compilation date?

Golang malware has become very popular over the past few years.

<p>
<details>
<summary> example </summary>
<pre>
1. Sandworm-Team (Exaramel-Linux)
2. APT28 Zebrocy
3. Ekans
4. Kinsing (miner)
5. Mustang Panda 
6. Rocke
7. APT29 WellMess/WellMail
8. Gobot
9. rocke
10. htran
11. goodor
12. FritzFrog
13. Gobrut
14. Geacon
15. Kaiji
16. NSPPS
17. Carbanak
18. Veil
19. AgeLocker
20. Notrobin
21. r2r2
22. Blackrota
23. IPStorm
24. RobbinHood
25. TinyBanker
26. CHAOS
27. NEsha
28. Hercules
29. Gandalf
30. RDW
31. hershell
32. ARCANUS
33. braincrypt
34. Klingon RAT
</pre>
</details>
</p>

The universality of the language allows you to compile the same source code for different architectures and operating systems, including Linux (as an ELF file).

I recently watched a video [Golang Malware: Using the attackers force against them](https://www.youtube.com/watch?v=kwrIr8Ydwro) which has an interesting point. The point is that it is possible to calculate an approximate timestamp of a binary using dependencies. The calculation algorithm from the video is as follows:

1. We take golang ELF malware
2. Get a list of dependencies
3. Get the dependency version
4. Get the date of a specific version (release) of the dependency
5. Create a list of dates
6. We take the latest date, it will be the approximate (minimum) timestamp

## Code

I decided to implement this idea in the code. The [gore](github.com/goretk/gore) project was used to get the dependencies. Keep in mind that dependencies can only be obtained from a binary if it has a [gopclntab](https://www.mandiant.com/resources/golang-internals-symbol-recovery) section. We will consider the Kinsing malware sample as an example. [Kinsing](https://attack.mitre.org/software/S0599/) is a golang linux malware that runs the miner. gore returns a list of dependencies like this:

<pre>
\go\pkg\mod\github.com\armon\go-socks5@v0.0.0-20160902184237-e75332964ef5
\go\pkg\mod\github.com\tklauser\go-sysconf@v0.3.10
\go\pkg\mod\github.com\shirou\gopsutil@v3.21.11+incompatible\mem
\go\pkg\mod\github.com\kelseyhightower\envconfig@v1.4.0
\go\pkg\mod\github.com\tklauser\numcpus@v0.4.0
\go\pkg\mod\github.com\shirou\gopsutil@v3.21.11+incompatible\process
\go\pkg\mod\github.com\asaskevich\govalidator@v0.0.0-20210307081110-f21760c49a8d
\go\pkg\mod\github.com\google\btree@v1.0.1
\go\pkg\mod\github.com\shirou\gopsutil@v3.21.11+incompatible\cpu
\go\pkg\mod\github.com\hashicorp\yamux@v0.0.0-20211028200310-0bc27b27de87
\go\pkg\mod\github.com\peterbourgon\diskv@v2.0.1+incompatible
\go\pkg\mod\github.com\go-resty\resty\v2@v2.7.0
\go\pkg\mod\github.com\op\go-logging@v0.0.0-20160315200505-970db520ece7
\go\pkg\mod\golang.org\x\net@v0.0.0-20220225172249-27dd8689420f\context
\go\pkg\mod\golang.org\x\net@v0.0.0-20220225172249-27dd8689420f\publicsuffix
\go\pkg\mod\github.com\nu7hatch\gouuid@v0.0.0-20131221200532-179d4d0c4d8d
\go\pkg\mod\github.com\shirou\gopsutil@v3.21.11+incompatible\host
\go\pkg\mod\github.com\shirou\gopsutil@v3.21.11+incompatible\net
\go\pkg\mod\github.com\kardianos\osext@v0.0.0-20190222173326-2bc1f35cddc0
\go\pkg\mod\github.com\paulbellamy\ratecounter@v0.2.0
\go\pkg\mod\golang.org\x\sys@v0.0.0-20220319134239-a9b59b0215f8\unix
\go\pkg\mod\github.com\shirou\gopsutil@v3.21.11+incompatible\internal\common
</pre>

If you look closely, you'll find that Golang represents dependency versions in three ways:

1.v0.0.0-20160902184237-e75332964ef5
2.v3.21.11+incompatible\cpu
3.v0.3.10

In the first form, the version of the dependency is always v0.0.0, that is, the dependency has no releases, which means that the version string itself contains the creation date. Therefore, we simply take the date from the string.

In the second form, `incompatible` means that the dependency does not have a `go.mod` file and has a version greater than 2. We discard the word `incompatible` and take only the version.

In the third form, we have nothing but a version, so we move on to the next step.

Knowing only the version of the dependency, we can try to get the release date using the [golang library for working with GithubAPI](https://github.com/google/go-github). Let's look at the dependency `\go\pkg\mod\github.com\tklauser\go-sysconf@v0.3.10` as an example.
Since not all repositories have releases but only tags, the [`ListTags`](https://docs.github.com/en/rest/git/tags#get-a-tag) method is used. `owner` and `repo` in this case will be `tklauser` and `go-sysconf`

```go
tags, resp, err := client.Repositories.ListTags(context.Background(), owner, repo, nil)
```

The response has info about each tag (release)

![](/assets/images/golang_timestamp/1.png)

The tag has a `commit` field with a `SHA` field. Using the Github Commit API we get commit information

```go
commit, resp, err := client.Repositories.GetCommit(context.Background(), owner, repo, sha, nil)
```

The response will contain the date of the commit.

![](/assets/images/golang_timestamp/2.png)

Now we have specific date for a specific dependency. We find the latest date among result. This date is the approximate (minimum) timestamp of the sample:

![](/assets/images/golang_timestamp/3.png)

[**Source Code Link**](https://github.com/thatskriptkid/chrononz)