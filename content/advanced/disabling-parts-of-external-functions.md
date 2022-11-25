---
title: "Disabling Parts of External Functions"
author: Aragas
date: 2022-11-25T01:30:00+03:00
---

# Introduction
Sometimes you need to use a game function, but with some parts disabled. For example, `Clan.OnCompanionAdded` not only adds a hero to the clan companion list, 
but also adds him to the hero list.
```csharp
public void OnCompanionAdded(Hero companion)
{
  _companionsCache.Add(companion);
  OnHeroAdded(companion); // Function that adds the hero to the hero list
}
```
But what if we don't want this? One way you could just copy the function and get the maintenance burden of it.  
Another way would be to use a Harmony reverse patch. You can get a copy of the function and manually manipulate the IL code to disable the `OnHeroAdded` call.  
We suggest a higher level approach, based on `using` scopes and `IDisposable`.  

## Scoped Disabling Technique
`using` allows us to create a scope. While this scope is active, we use a harmony patch that disables the execution of the unwanted function.
```csharp
using (new AddCompanionActionHandler())
{
    // Within this scope any call to `OnHeroAdded` will be skipped.
    oldLeader.Clan.OnCompanionAdded(oldLeader);
}
```
The Harmony patch
```csharp
public class AddCompanionActionHandler : IDisposable
{
    public AddCompanionActionHandler() => AddCompanionActionPatch.SkipChange = true;
    public void Dispose() => AddCompanionActionPatch.SkipChange = false;
}

internal class AddCompanionActionPatch
{
    internal static bool SkipChange = false;

    public static bool Enable(Harmony harmony)
    {
        return true &
            harmony.TryPatch(
                original: AccessTools2.Method(typeof(Clan), "OnHeroAdded"),
                prefix: AccessTools2.Method(typeof(AddCompanionActionPatch), nameof(Prefix)));
    }

    private static bool Prefix() => !SkipChange;
}
```
