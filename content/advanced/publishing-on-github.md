---
title: "Publishing Mods on GitHub Actions (NexusMods, Steam Workshop, NuGet)"
author: Aragas
date: 2022-11-29T19:30:00+03:00
---

# Introduction
We created some easy to use reusable workflows that you can use to automatically publish your mods on NexusMods, Steam Workshop, NuGet/GPR and GitHub Releases.  
The workflow is a multijob.  
Here's a [full example](https://github.com/BUTR/Bannerlord.UIExtenderEx/blob/9a6f063ec9f8924feb51294ff0056501efc863c5/.github/workflows/publish.yml).  
First, you need to build your module on GitHub Actions and publish the binaries as an artifact.  

Here's an abstract example that uses [Bannerlord.BuildResources](https://github.com/BUTR/Bannerlord.BuildResources)
for automatically building the mod with the proper structure.
```yml
  build-module:
    name: Build Module
    runs-on: ubuntu-latest
    steps:
    - name: Checkout Repository
      uses: actions/checkout@v2
        
    - name: Setup .NET 6
      uses: actions/setup-dotnet@master
      with:
        dotnet-version: 6.0.x

    - name: Build Bannerlord.Module
      run: >-
        mkdir bannerlord;
        dotnet build src/Bannerlord.Module/Bannerlord.Module.csproj --configuration Release -p:GameFolder="$PWD/bannerlord";
      shell: pwsh

    - name: Upload Bannerlord folder
      uses: actions/upload-artifact@v3
      with:
        name: bannerlord
        path: ./bannerlord/
```

# NexusMods
NexusMods will require you to set the proper ModId (`nexusmods_mod_id`), this is the number at the end of your mod url.  
Also, it will require to set 2 secrets, `NEXUSMODS_APIKEY` and `NEXUSMODS_COOKIES`.
You can get the ApiKey from [here](https://www.nexusmods.com/users/myaccount?tab=api). The Cookies will require you to log in to NexusMods via Firefox.
F12 and go to Network tab. Find any XHR method to nexusmods.com and copy the Cookies header. This is our NEXUSMODS_COOKIES secret.
```yml
  publish-on-nexusmods:
    needs: ["build-module"]
    uses: BUTR/workflows/.github/workflows/release-nexusmods.yml@master
    with:
      nexusmods_game_id: mountandblade2bannerlord
      nexusmods_mod_id: 0
      mod_filename: Bannerlord.Module
      mod_version: INJECT YOUR VERSION SOMEHOW
      mod_description: INJECT YOUR CHANGELOG SOMEHOW
      artifact_name: bannerlord
    secrets:
      NEXUSMODS_APIKEY: ${{ secrets.NEXUSMODS_APIKEY }}
      NEXUSMODS_COOKIES: ${{ secrets.NEXUSMODS_COOKIES }}
```

# Steam
If Steam Guard is used, you need the STEAM_AUTH_CODE secret. Follow this url as a [guide](https://github.com/SteamTimeIdler/stidler/wiki/Getting-your-%27shared_secret%27-code-for-use-with-Auto-Restarter-on-Mobile-Authentication#getting-shared-secret-from-ios-windows):
You need to either switch to Steam Desktop Authenticator or extract the Shared Key secret from your phone's files.
If Steam Guard is not used, `STEAM_LOGIN` and `STEAM_PASSWORD` secrets will be enough.

```yml
  publish-on-steam:
    needs: ["build-module"]
    uses: BUTR/workflows/.github/workflows/release-steam.yml@master
    with:
      workshop_id: 0
      mod_id: Bannerlord.Module
      mod_description: INJECT YOUR CHANGELOG SOMEHOW
      artifact_name: bannerlord
    secrets:
      STEAM_LOGIN: ${{ secrets.STEAM_WORKSHOP_LOGIN }}
      STEAM_PASSWORD: ${{ secrets.STEAM_WORKSHOP_PASSWORD }}
      STEAM_AUTH_CODE: ${{ secrets.STEAM_WORKSHOP_AUTH_CODE }}
```

# NuGet/GRP
You'll need an API Key for NuGet.
```yml
  publish-on-nuget:
    needs: ["build-module"]
    uses: BUTR/workflows/.github/workflows/release-nuget.yml@master
    with:
      project_path: src/Bannerlord.Module/Bannerlord.Module.csproj
    secrets:
      NUGET_API_KEY: ${{ secrets.NUGET_API_KEY }}
```


# GitHub Releases
```yml
  publish-on-github:
    needs: ["build-module"]
    uses: BUTR/workflows/.github/workflows/release-github.yml@master
    with:
      mod_id: Bannerlord.Module
      mod_version: INJECT YOUR VERSION SOMEHOW
      mod_description: INJECT YOUR CHANGELOG SOMEHOW
      artifact_name: bannerlord
```
