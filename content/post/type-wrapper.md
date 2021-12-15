---
author: Aragas
title: "Type Wrapper"
date: 2021-12-15T17:57:01+09:00
linktitle: Type Wrapper
menu:
  main:
    parent: Advanced
weight: 10
---

# Introduction
Our BUTR mods have a famous history of backwards compatibility.  
We use several techniques to achieve this without causing huge performance drops by using Harmony's built in reflection tools and caching.  
One of the techniques we use is type wrapping.  
It's used for several cases:
* When types are moved between assemblies. Generally, to fix this you need to recompile your mod with the new locations, but it won't be backwards compatible this way. Instead, we stop working with the type directly and wrap it as an object, exposing an abstraction to work with.
* Incompatible signature changes. The method signature was replaced without keeping the old signature. Depending on the game version, you'll need to call different methods.

## Type Move

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
