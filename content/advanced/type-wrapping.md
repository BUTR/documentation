---
title: "Type Wrapping"
author: Aragas
date: 2021-12-15T17:57:01+03:00
---

# Introduction
Our BUTR mods have a famous history of backwards compatibility.  
We use several techniques to achieve this without causing huge performance drops by using Harmony's built in reflection tools and caching.  
One of the techniques we use is type wrapping.  
It's used for several cases:
* When types are moved between assemblies. Generally, to fix this you need to recompile your mod with the new locations, but it won't be backwards compatible this way. Instead, we stop working with the type directly and wrap it as an object, exposing an abstraction to work with.
* Incompatible signature changes. The method signature was replaced without keeping the old signature. Depending on the game version, you'll need to call different methods.

## Type Move
Read-life example of type move.  
In our BUTRLoader, we target e1.5.0 as the minimal supported version. Since the type was moved in e1.5.x, we either need to have 2 compiled assemblies and load one depending on the game version (see Implementation Loading), or create a type wrapper, what we did.
```csharp
// The game at some point moved type ModuleInfo
// TaleWorlds.Library.ModuleInfo
// to
// TaleWorlds.ModuleManager.ModuleInfo
// The type can't be directly referenced and used because it's location
// is not consistent anymore/
// So we instead create a wrapper class that access the type indirectly

internal sealed class ModuleInfoWrapper
{
    private delegate string GetIdDelegate(object instance);
    private delegate string GetAliasDelegate(object instance);
    private delegate bool GetIsSelectedDelegate(object instance);

    private static readonly Type? OldModuleInfoType = AccessTools2.TypeByName("TaleWorlds.Library.ModuleInfo");
    private static readonly Type? NewModuleInfoType = AccessTools2.TypeByName("TaleWorlds.ModuleManager.ModuleInfo");
    public static readonly Type? ModuleInfoType = OldModuleInfoType ?? NewModuleInfoType;

    private static readonly GetIdDelegate? GetId = AccessTools2.GetPropertyGetterDelegate<GetIdDelegate>(ModuleInfoType, "Id");
    private static readonly GetAliasDelegate? GetAlias = AccessTools2.GetPropertyGetterDelegate<GetAliasDelegate>(ModuleInfoType, "Alias");
    private static readonly GetIsSelectedDelegate? GetIsSelected = AccessTools2.GetPropertyGetterDelegate<GetIsSelectedDelegate>(ModuleInfoType, "IsSelected");

    public static ModuleInfoWrapper Create(object? @object) => new(@object);

    public string Id => _id ??= Object is null ? string.Empty : GetId?.Invoke(Object) ?? string.Empty;
    private string? _id;
    public string Alias => _alias ??= Object is null ? string.Empty : GetAlias?.Invoke(Object) ?? string.Empty;
    private string? _alias;
    public bool IsSelected => Object is null ? false : GetIsSelected?.Invoke(Object) ?? false;

    public object? Object { get; }

    private ModuleInfoWrapper(object? @object)
    {
        Object = @object;
    }
}
```

We use the info from an already existing type `LauncherModuleVM`. It has a property `ModuleInfo Info`. Since we can't access the property directly, we create another wrapper and access the field via the wrapper.
```csharp
internal sealed class LauncherModuleVMWrapper
{
    private static readonly Type? LauncherModuleVMType = AccessTools2.TypeByName("TaleWorlds.MountAndBlade.Launcher.LauncherModuleVM");
    private static readonly AccessTools.FieldRef<object, object>? GetInfo = AccessTools2.FieldRefAccess<object>(LauncherModuleVMType!, "Info");

    public static LauncherModuleVMWrapper Create(object @object) => new(@object);

    public ModuleInfoWrapper Info => _info ??= ModuleInfoWrapper.Create(GetInfo?.Invoke(Object));
    private ModuleInfoWrapper? _info;

    public object Object { get; }

    private LauncherModuleVMWrapper(object @object)
    {
        Object = @object;
    }
}
```
And then access it like this
```csharp
LauncherModuleVM targetModule = obj.Module;
LauncherModuleVMWrapper targetModuleWrapper = LauncherModuleVMWrapper.Create(targetModule);
ModuleInfoWrapper targetModuleInfoWrapper = targetModuleWrapper.Info;
string targetModuleInfoId = targetModuleInfoWrapper.Id
```

## Signature Change
Read-life example of signature change.
```csharp
// The game at some point changed GauntletLayer's constructor from
// (int localOrder, string categoryId = "GauntletLayer")
// to
// (int localOrder, string categoryId = "GauntletLayer", bool shouldClear = false)
// It introduces an optional parameter. It might seem that such a change is backward compatible,
// but it's not ABI compatible at all.

// Some ScreenBase derived class
protected override void OnInitialize()
{
    base.OnInitialize();
    _dataSource = new EditValueVM(_settingProperty);
    _gauntletLayer = = new GauntletLayer(4000, "GauntletLayer"); // Broken
    _gauntletMovie = LoadMovie is not null ? LoadMovie(_gauntletLayer, "EditValueView_MCM", _dataSource) : null; // ignore for now
    _gauntletLayer.Input.RegisterHotKeyCategory(HotKeyManager.GetCategory("ChatLogHotKeyCategory"));
    _gauntletLayer.InputRestrictions.SetInputRestrictions(true, InputUsageMask.All);
    _gauntletLayer.IsFocusLayer = true;
    AddLayer(_gauntletLayer);
    ScreenManager.TrySetFocus(_gauntletLayer);
}
```

To fix this, you create a type wrapper
```csharp
internal static class GauntletLayerWrapper
{
    // The first version of the signature
    private delegate GauntletLayer V1Delegate(int localOrder, string categoryId = "GauntletLayer");
    // THe second version
    private delegate GauntletLayer V2Delegate(int localOrder, string categoryId = "GauntletLayer", bool shouldClear = false);

    private static readonly V1Delegate? V1;
    private static readonly V2Delegate? V2;

    static GauntletLayerWrapper()
    {
        // Iterate over each constructor
        // You could manually find each constructor instead
        // But in our opinion one full manual iteration is more effectice
        foreach (var constructorInfo in HarmonyLib.AccessTools.GetDeclaredConstructors(typeof(GauntletLayer), false))
        {
            var @params = constructorInfo.GetParameters();
            switch (@params.Length)
            {
                case 2:
                    V1 = AccessTools2.GetDelegate<V1Delegate>(constructorInfo);
                    break;
                case 3:
                    V2 = AccessTools2.GetDelegate<V2Delegate>(constructorInfo);
                    break;
            }
        }
    }

    // The new constructor wrapper
    public static GauntletLayer? Create(int localOrder, string categoryId = "GauntletLayer", bool shouldClear = false)
    {
        if (V1 is not null)
            return V1(localOrder, categoryId);
        if (V2 is not null)
            return V2(localOrder, categoryId, shouldClear);
        return null;
    }
}
```

You can use it like this
```csharp
protected override void OnInitialize()
{
    base.OnInitialize();
    _dataSource = new EditValueVM(_settingProperty);
    // Since the signature might change a third time, check that the result is not null
    if (GauntletLayerUtils.Create(4000, "GauntletLayer") is { } gauntletLayer)
    {
        _gauntletLayer = gauntletLayer;
        _gauntletMovie = LoadMovie is not null ? LoadMovie(_gauntletLayer, "EditValueView_MCM", _dataSource) : null; // ignore for now
        _gauntletLayer.Input.RegisterHotKeyCategory(HotKeyManager.GetCategory("ChatLogHotKeyCategory"));
        _gauntletLayer.InputRestrictions.SetInputRestrictions(true, InputUsageMask.All);
        _gauntletLayer.IsFocusLayer = true;
        AddLayer(_gauntletLayer);
        ScreenManager.TrySetFocus(_gauntletLayer);
    }
}
```
