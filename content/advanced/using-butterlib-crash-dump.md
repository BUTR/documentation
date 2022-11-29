---
title: "Using ButterLib's Crash Dump"
author: Aragas
date: 2022-10-23T17:12:00+03:00
---


## Introduction
Both the Vanilla Game and ButterLib provide [dump (.dmp) files](https://en.wikipedia.org/wiki/Core_dump) that modders can use to open in Visual Studio and see both the stacktrace, threads and stack variables to get more info about a crash.

## How to Get a Dump
The game creates a dump file at **C:\ProgramData\Mount and Blade II Bannerlord\crashes** after agreeing with the game prompt to send to the developers the crash report.
The dump is fairly big, because it has a lot of flags by default:
```
MiniDumpWithDataSegs | MiniDumpWithHandleData | MiniDumpScanMemory | MiniDumpWithUnloadedModules | MiniDumpWithIndirectlyReferencedMemory | MiniDumpWithProcessThreadData | MiniDumpWithFullMemoryInfo | MiniDumpWithThreadInfo | MiniDumpWithFullAuxiliaryState | MiniDumpIgnoreInaccessibleMemory | MiniDumpWithTokenInformation | MiniDumpWithModuleHeaders
```
ButterLib generates a dump file inside the crash report only when the report is saved locally as a file.
When the html file is opened, a button appears `Get Minidump` that will save the dump as a separate file. Out flags are:
```
MiniDumpFilterModulePaths | MiniDumpWithIndirectlyReferencedMemory | MiniDumpScanMemory
```

## Analyzing the Dump
After opening the dump in Visual Studio, you won't get any useful data first.
This is because we need to start a debug session (Debug with Managed Only) and show VS where both the .dll and .pdb (symbols) files are.  
![https://media.discordapp.net/attachments/423816778276274176/1033743972226580490/unknown.png](https://media.discordapp.net/attachments/423816778276274176/1033743972226580490/unknown.png)

The game doesn't provide the .pdb files, but they can be created by ILSpy and JetBrains dotPeek.
Open the .dll files there and when right clicking on the file a prompt about creating a .pdb file should appear.
![https://media.discordapp.net/attachments/423816778276274176/1033744427677659367/unknown.png](https://media.discordapp.net/attachments/423816778276274176/1033744427677659367/unknown.png)

After that, link VS to them and you should have a standard exception window with the source code!
![https://media.discordapp.net/attachments/423816778276274176/1033744073376419921/unknown.png](https://media.discordapp.net/attachments/423816778276274176/1033744073376419921/unknown.png)
