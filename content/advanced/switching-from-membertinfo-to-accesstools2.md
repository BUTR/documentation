---
title: "Switching from MemberInfo to AccessTools2"
author: Aragas
date: 2021-12-15T17:57:01+03:00
---

# Introduction
Using any of the `MemberInfo` types is slow. Especially when they are not cached.  
A better way for accessing type members is to create delegates, with them the access speed is on pair with direct access.  
Harmony created a helper class `AccessTools` for those cases and BUTR created an extended version of it.  
In this article we'll show how you can speed up your code.

## Caching
Always cache the reflection if it's gonna be called multiple times.

## Accessing fields
```csharp
// Usually it's done like this:
private static readonly FieldInfo? Info = AccessTools2.Field(typeof(LauncherModuleVM), "Info");
// And then invoked like this:
LauncherModuleVM obj = ...; // the LauncherModuleVM instance
if (Info is not null)
{
    string str = Info.GetValue(obj);
    Info.SetValue(obj, "newstr");
}


// Instead use AccessTools.FieldRef
private static readonly AccessTools.FieldRef<string, LauncherModuleVMType>? Info = AccessTools2.FieldRefAccess<string, LauncherModuleVMType>("Info");

LauncherModuleVM obj = ...; // the LauncherModuleVM instance
if (Info is not null)
{
    string str = Info(obj);
    Info(obj) = "newstr";
}
```
## Calling methods
```csharp
// Usually it's done like this:
private static readonly MethodInfo? Method = AccessTools2.Method(typeof(LauncherModuleVM), "Method");
// And then invoked like this:
LauncherModuleVM obj = ...; // the LauncherModuleVM instance
if (Method is not null)
{
    bool b = Info.Invoke(obj, new object[] {});
}


// Instead use AccessTools.FieldRef
private delegate bool MethodDelegate(LauncherModuleVM instance);

private static readonly MethodDelegate Method = AccessTools2.GetDelegate<MethodDelegate>(typeof(LauncherModuleVM), "Method");

LauncherModuleVM obj = ...; // the LauncherModuleVM instance
if (Method is not null)
{
    bool b = Info(obj);
}
```

## Accessing properties
```csharp
// Usually it's done like this:
private static readonly PropertyInfo? Name = AccessTools2.Property(typeof(LauncherModuleVM), "Name");
// And then invoked like this:
LauncherModuleVM obj = ...; // the LauncherModuleVM instance
if (Name is not null)
{
    bool b = Info.GetValue(obj);
    Info.SetValue(obj, true);
}


// Instead use AccessTools.FieldRef
private delegate bool GetNameDelegate(LauncherModuleVM instance);
private delegate void SetNameDelegate(LauncherModuleVM instance, bool @value);

private static readonly GetNameDelegate GetName = AccessTools2.GetPropertyGetterDelegate<GetNameDelegate>(typeof(LauncherModuleVM), "Name");
private static readonly SetNameDelegate SetName = AccessTools2.GetPropertySetterDelegate<SetNameDelegate>(typeof(LauncherModuleVM), "Name");

LauncherModuleVM obj = ...; // the LauncherModuleVM instance
if (GetName is not null && SetName is not null)
{
    bool b = GetName(obj);
    SetName(obj, true);
}
```
