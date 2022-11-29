---
title: "Advanced Stacktrace Analytics of Crash Reports"
author: Aragas
date: 2022-10-21T23:45:00+03:00
---

# Introduction
ButterLib v2.2.4 added a new entry in its Crash Report section called **Enhanced Stacktrace**.  
It provides more info about the stacktrace of the exception -every frame (method call) provides more info:  

* If the method is changed by Harmony, it shows the original method and any prefix, postfix, transpiler or finalizer that is added, included with the Module that introduces the patch;  
* An IL Offset is added;  

# IL Offsets
You should be able to know now the line that caused the exception, even if no debug symbols were present at the time of the crash.  
This will be the most helpful when debugging vanilla game code.  
Here's how to use it:  

* Download dnSpy or any other software that can show the IL code of a method;  
* Convert the IL Offset to hexadecimal;  
* Open the IL Code of the method;  
* The IL Offset will link to the exact line the code threw an exception;  

# Example
As an example, here's a [crash report](https://report.butr.link/8EA037.html). The IL Offset of the method **CalculateSettlementValueForFaction** is **73** (**0x49** hex).  
![https://media.discordapp.net/attachments/423816778276274176/1032936619822567434/unknown.png](https://media.discordapp.net/attachments/423816778276274176/1032936619822567434/unknown.png)
The IL code shows that at **0049** is the line  
```csharp
itemObject.ItemHolsters = (string[])craftedData.Template.ItemHolsters.Clone();
```
![https://media.discordapp.net/attachments/423816778276274176/1032936863205433414/unknown.png](https://media.discordapp.net/attachments/423816778276274176/1032936863205433414/unknown.png)
On which the NullReferenceException occured.
