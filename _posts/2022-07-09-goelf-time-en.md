---
layout: post
title: Estimating a minimum timestamp for Go ELF malware
tags: [linux, malware analysis]
category: [en]
---

## Intro

A timestamp, or compilation date, is an important characteristic of a malware sample. It can indicate whether the sample is new, whether it may have appeared in earlier attacks, and roughly when an operation was prepared. But what can we do when the sample is an ELF binary, a format that does not store a compilation timestamp?

Malware written in Go has become increasingly common in recent years.

<p>
<details>
<summary>Examples</summary>
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

Go's cross-platform toolchain makes it easy to compile the same source code for different operating systems and architectures, including Linux, where the output is an ELF binary.

I recently watched [Golang Malware: Using the Attackers' Force Against Them](https://www.youtube.com/watch?v=kwrIr8Ydwro), which presents an interesting idea: a Go binary's dependencies can be used to estimate its earliest possible build date. The process described in the video is:

1. Take a malicious Go ELF binary.
2. Extract its dependency list.
3. Identify each dependency version.
4. Find the release date of each version.
5. Collect those dates.
6. Use the most recent date as the binary's approximate minimum timestamp.

## Code

I implemented this idea in code, using [GoRE](https://github.com/goretk/gore) to extract dependencies. Keep in mind that GoRE can recover them only when the binary contains a [`gopclntab`](https://www.mandiant.com/resources/golang-internals-symbol-recovery) section. As an example, we will use a [Kinsing](https://attack.mitre.org/software/S0599/) sample, a Linux malware family written in Go that deploys a cryptocurrency miner. GoRE returns a dependency list like this:

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

The list contains three forms of Go module versions:

1. `v0.0.0-20160902184237-e75332964ef5`

2. `v3.21.11+incompatible\cpu`

3. `v0.3.10`

The first form is a Go pseudo-version. Its timestamp is embedded directly in the version string, so we can extract the date from it.

In the second form, the `+incompatible` suffix indicates that the module has a major version of 2 or higher but does not follow the module path convention introduced with Go modules. We remove the suffix and any package subpath, retaining only the version.

The third form contains only a semantic version, so its date must be resolved separately.

Given only a dependency version, we can look up its date with the [Go client library for the GitHub API](https://github.com/google/go-github). Consider `\go\pkg\mod\github.com\tklauser\go-sysconf@v0.3.10` as an example.

Not every repository publishes GitHub Releases, but version tags are usually present, so the code uses [`ListTags`](https://docs.github.com/en/rest/git/tags#get-a-tag). In this example, `owner` is `tklauser` and `repo` is `go-sysconf`.

```go
tags, resp, err := client.Repositories.ListTags(context.Background(), owner, repo, nil)
```

The response contains information about each tag.

![](/assets/images/golang_timestamp/1.png)

Each tag includes a `commit` field containing a SHA. We use that SHA to retrieve commit metadata through the GitHub API.

```go
commit, resp, err := client.Repositories.GetCommit(context.Background(), owner, repo, sha, nil)
```

The response includes the commit date.

![](/assets/images/golang_timestamp/2.png)

After resolving a date for every dependency, we select the latest one. The sample could not have been built before that dependency existed, so this date is its approximate minimum timestamp:

![](/assets/images/golang_timestamp/3.png)

[**Source code**](https://github.com/thatskriptkid/chrononz)
