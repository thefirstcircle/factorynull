---
layout: post
title:  "Finding WSD Printers with Powershell"
author: "Steve O'Neill"
comments: true
categories: sysadmin
tags: windows
---

WSD, or *[Web Services for Devices](https://techcommunity.microsoft.com/t5/ask-the-performance-team/ws2008-the-wsd-port-monitor/ba-p/372760),* is a printer port monitor that was developed by Microsoft around the time Vista was released. It doesn’t have a good reputation with system administrators - I’ve heard some try to disable it in the firmware of every printer that gets purchased in their company - but it seems to have applications that are useful outside of the enterprise.

Either way, you might find it useful to audit the WSD printers in your organization someday. Maybe you have just added a print server (or started using [Universal Print](https://docs.microsoft.com/en-us/universal-print/fundamentals/universal-print-whatis)). Or maybe you are merging with another department and want to do some discovery on what you need to start managing.

Here are a few lines of Powershell I wrote after not finding anything else online:

```powershell
Get-ChildItem -Path Registry::\HKLM\SYSTEM\CurrentControlSet\Enum\SWD\DAFWSDProvider -ErrorAction SilentlyContinue | Get-ItemProperty | Select-Object LocationInformation,Mfg,FriendlyName | Sort-Object -Property LocationInformation,Mfg,FriendlyName | Get-Unique -AsString
```

Here is the output:

```powershell
LocationInformation Mfg FriendlyName
------------------- --- ------------
http://10.0.0.157:3911/ HP HP3D9932 (HP OfficeJet Pro 6960)
http://192.168.1.142:3911/ HP HP69791D.hsd1.ma.comcast.net (HP OfficeJet Pro 8020 series)
```

As shown above, querying for the IP addresses of WSD printers is not as simple as getting the IPs of regular TCP/IP printers, which can be viewed by the following simple command:

```powershell
Get-Printer | select Name,PortName,DriverName
```

If you have a way of deploying this script to users’ computers, you can get some pretty interesting reporting of what network printers have been installed locally. Just keep in mind that home printers may be included in whatever results come back - so stay respectful of privacy.