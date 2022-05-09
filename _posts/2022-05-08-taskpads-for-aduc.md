---
layout: post
title:  "Using RSAT Tools with Custom Taskpads"
author: "Steve O'Neill"
comments: true
categories: sysadmin
tags: windows
---

On-premises Active Directory is certainly going away. Soon it will be rare to find RSAT tools installed on any IT person’s laptop [or... maybe not?].

Looking back, I will miss a feature of the Users and Computers MMC snap-in that always seemed relatively obscure: the [“taskpad”](https://petri.com/add-taskpad-custom-mmc/).

The premise is simple. Taskpads added clickable, user defined options to the already-familiar ADUC management interface. 

Taskpads aren’t specific to the ADUC snap-in, but they can be pointed to batch files - and, yes, Powershell - for any special function you want.

Behold my custom taskpad, circa 2018:

![Untitled]({{site.url}}/docs/assets/img/taskpad%20f1162ea308774fb3bb74bf699eec8984/Untitled.png)

To create your own taskpad view, open a CMD window, open ```mmc```, and use ```ctl+m``` to open the Add or Remove Snap-ins dialog. Add ‘Active Directory Users and Computers’ to the Console Root. Then find an OU or object in AD, right-click, and select ‘New Taskpad View’. Follow the wizard and use the defaults.

## Scripting

A “Shell command” task is capable of passing whichever parameter you specify into your script as an argument:

![Untitled]({{site.url}}/docs/assets/img/taskpad%20f1162ea308774fb3bb74bf699eec8984/Untitled%201.png)

Here, $COL<0> is ‘**Name**’. ‘Name’ can apply to both computer and user objects. 

If you click the arrow to the right of the box, it will show you options for other parameters, like Email Address, Department, Zip Code, etc. Not so useful for computer objects, but maybe they have a place somewhere.

Let’s look at ping_computer.bat, the batch file I am pointing this task towards:

```powershell
@echo off

title PING [%1]
color F0
ping.exe %1 -4 -t
pause
```

So - the title is ‘PING’ with the variable passed in as %1. The color of the box is white, and ping.exe is attempting to reach the computer until it’s cancelled by the user (```-t```) and using IPv4 only (```-4```). Fairly easier than typing the computer’s name into a command prompt, and just one click to accomplish now:

![Untitled]({{site.url}}/docs/assets/img/taskpad%20f1162ea308774fb3bb74bf699eec8984/Untitled%202.png)

## URL string insertion

You may use a web service - say, for asset management - that supports URL-based navigation. For example, here is the “Look up in KACE” task (Lookup_in_KACE.bat):

```powershell
powershell -Command "& {start-process  "msedge.exe" -argumentlist ""https://myKACEinstace.com/adminui/computer_inventory.php?LABEL_ID=&SEARCH_SELECTION="%1"""}"
```

Crudely or not, this method passes in the ‘Name’ variable (represented as %1) and results in the computer showing up in a browser with just one click:

![Untitled]({{site.url}}/docs/assets/img/taskpad%20f1162ea308774fb3bb74bf699eec8984/Untitled%203.png)

The “Look up in ServiceNow” task uses the same method: 

```powershell
powershell -Command "& {start-process  "msedge.exe" -argumentlist ""https://myinstance.service-now.com/nav_to.do?uri=textsearch.do?sysparm_search="%1"""}"
```

And the result, in one click:

![Untitled]({{site.url}}/docs/assets/img/taskpad%20f1162ea308774fb3bb74bf699eec8984/Untitled%204.png)

By using ServiceNow’s Javascript [GlideSystem](https://docs.servicenow.com/bundle/rome-platform-user-interface/page/use/navigation/reference/r_NavigatingByURLExamples.html), you can even create a new ticket with the parameterized input. Here is part of the “Decommission” task:

```powershell
set URL="https://umass.service-now.com/nav_to.do?uri=incident.do?sys_id=-1%26sysparm_query=priority=1^incident_state=3^caller_id=javascript:gs.getUserID()^short_description=Decommissioned"

powershell -Command "& {start-process  "msedge.exe" -argumentlist "%URL%""}"

REM See https://docs.servicenow.com/bundle/rome-platform-user-interface/page/use/navigation/reference/r_NavigatingByURLExamples.html

pause
```

And its result - see how it auto-filled the description and added me as the name of caller?

![Untitled]({{site.url}}/docs/assets/img/taskpad%20f1162ea308774fb3bb74bf699eec8984/Untitled%205.png)

## Conclusion

If you don't have access to better methods, this could be a weird and wacky way of making some routine tasks slightly more... tolerable. You are really just limited by your imagination!