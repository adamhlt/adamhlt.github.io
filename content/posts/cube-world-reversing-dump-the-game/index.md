---
title: "Cube World Reversing - Unpack the game"
date: 2022-02-15T17:54:41+02:00
draft: false
tags: [Game Hacking, Reverse Engineering]
summary: "How to unpack Cube World, a game packed with Steam DRM, to reverse it and make a cheat on it."
series: ["Cube World Reversing"]
series_order: 1
---

{{< alert cardColor="#e63946" iconColor="white" textColor="white">}}
I don't give any information about game cracking, I have a legitimate version of Cube World (v1.0) on Steam and that's why I need to remove Steam DRM to analyze the game.
{{< /alert >}}

## Why Cube World ?

[Cube World](https://store.steampowered.com/app/1128000/Cube_World/) is an action role-playing game developed and published by Picroma. The alpha version was released in 2013, today the official version is available on Steam. I chose this game because there is not much information about it in the game hacking community, you can find some `Cheat Engine` tables but not more. That's why I chose this game, it forces me to do everything by myself and in my opinion it's a great way to train.

The aim of this series is to reverse the game and implement some cheat functions like god mode, fly hack, speed hack...
The game Cube World uses its own game engine which is also called Cube World, that's why it can be a great reverse engineering challenge to reverse it.

## How to get started ?

Before we start to reverse the game, we need some tools:
- [Cheat Engine](https://www.cheatengine.org)
- [x64dbg](https://x64dbg.com)
- [ReClass.NET](https://github.com/ReClassNET/ReClass.NET)
- [IDA Pro](https://hex-rays.com/ida-pro/) / [Ghidra](https://ghidra-sre.org)
- [PE Bear](https://github.com/hasherezade/pe-bear-releases)
- [Detect It Easy](https://github.com/horsicq/Detect-It-Easy)
- [Steamless](https://github.com/atom0s/Steamless)

Some of these tools will only be used in this part of the reversal, `PE Bear` and `Detect It Easy` are tools used to analyse the PE file of the game.

The first thing we want to do is load the game into `IDA` (or `Ghidra`) to look at the instructions and see if we can start analysing the game.

```
[steam_path]\steamapps\common\Cube World\cubeworld.exe
```

If you try to load the game into `IDA`, you will get errors like in the following screenshots (I think it is the same in Ghidra), this is not a good sign.

Also, if we look at the `.text` section, it is full of strange instructions and many functions are referenced as `nullsub` functions.

![Loading Error](https://user-images.githubusercontent.com/48086737/234102864-a7f784dd-46b5-4a8c-a646-d28aa2c68ecf.png "Error message when trying to load the game into IDA.")
![.text Section](https://user-images.githubusercontent.com/48086737/234102919-2e8612bd-1265-40b6-985c-0ad899b17f7f.png "IDA, with functions referenced as `nullsub` functions and `.text` section.")
![Section Zoom](https://user-images.githubusercontent.com/48086737/234102903-1a1c7bfc-ec87-4fb4-a502-9c78dabf35c7.png "Instructions in the `.text` section look strange and some of the functions cannot be analysed.")

We can take a quick look at the sections of the PE file of the game, we can see that there are 2 sections containing code, first the `.text` section which is normal, but we can see that the `.bind` section also contains code. This is a big sign that the game might be packed.

![Sections](https://user-images.githubusercontent.com/48086737/234103254-6dfc50b4-801c-4890-b2ff-0b88d622d6a0.png "Sections of the PE file of the game, view in IDA.")
![Entrypoint zoom](https://user-images.githubusercontent.com/48086737/234103277-2aae231d-3409-4349-b733-c731a60784b4.png "Entrypoint of the PE file of the game, in the `.bind` section.")

As you can see we have 2 sections of code and the entry point is in the `.bind` section, there are a lot of flags that the game can be packed, but to be sure and maybe identify the packer that is used we need to analyse the PE file of the game.

## Analysis of PE files

So the first thing to do is to use `PE Bear` and `Detect It Easy`, `PE Bear` helps us to see the structure of the PE file and confirm that the entry point is in the `.bind` section and `Detect It Easy` is a special tool that helps to identify packers and compilers.

![PE Bear](https://user-images.githubusercontent.com/48086737/234103386-3755cb8b-0ed4-4579-9cc2-dbeb3896d1f7.png "`PE Bear` of the game's PE file, it confirms that the entry point is in the `.bind` section.")

![Steam Stub](https://user-images.githubusercontent.com/48086737/234103414-5baef205-7c17-4e98-b0f3-8f13e9f7bc23.png "`Detect It Easy` identifies the packer as `Steam stub`.")

With `Detect It Easy`, we can now confirm that the game is packed with `Steam DRM`. So, to reverse the game and make our cheat, we need to unpack it first.

## Unpack the game

### Use Steamless
Now we know that the game comes with `Steam DRM`, you can find some information about the `DRM` [here](https://www.pcgamingwiki.com/wiki/User:Cyanic/Steam_DRM). The next step is to unpack the game, it is useless to do this manually as [Steamless](https://github.com/atom0s/Steamless) works great. `Steamless` is an automatic tool that allows you to remove the `Steam DRM` and decrypt the `.text` section of the game.

If you want to try unpacking the game manually, or want to know how `Steam DRM` works, you can look at the code of `Steamless`, every step of unpacking is described in the code.

![Steamless](https://user-images.githubusercontent.com/48086737/234103966-91ce7628-642e-4474-ab48-a84742876b89.png "`Steamless` home page.")

![Steamless Unpack](https://user-images.githubusercontent.com/48086737/234103987-5abf6a54-7e42-4b00-a498-6dd528eb957a.png "Unpack the game using `Steamless`.")

### Analysis of the unpacked file
As you can see it is very easy to unpack the game with `Steamless`, if you try to run the unpacked game it will crash as the new version of Cube World is very dependent on the `Steam API` and without patching functions the game will not work. In our case this is not important as we can do the analysis on the unpacked PE file with `IDA` and the debugging on the game that was dynamically unpacked by the `DRM` at runtime.

![New Sections](https://user-images.githubusercontent.com/48086737/234104204-079f8559-aebd-4cb3-829d-52f77f7f30ee.png "Sections of the unpacked PE file, view in IDA.")
![New .text Section](https://user-images.githubusercontent.com/48086737/234104226-86df20b1-c2f9-4b22-b248-de7fbe99792e.png "Decrypted `.text` section.")
![New Section Zoom](https://user-images.githubusercontent.com/48086737/234104237-bb8c0c69-05a8-42e1-94c2-d33826debef3.png "Clean instructions in the decrypted `.text` section.")

Now all the sections needed by the DRM have been removed, the `.text` section has been decrypted, the `IDA` analysis doesn't return any `nullsub` functions anymore and the analysis is much faster. The next step is to see if we can retrieve instructions from the debugger (`Cheat Engine` or `x64dbg`) into `IDA`.

## Setting up the debugger
### Using the VEH debugger
The last step is to set up our environment, if you try to use `Cheat Engine` with `Windows debugger`, Cube World will crash in a few cases, so I recommend to use `VEH debugger` with `Cheat Engine` (I never had a problem with `VEH`, but this is only for Cube World). In the following screenshots I show you how to configure `Cheat Engine`.

![VEH](https://user-images.githubusercontent.com/48086737/234104437-9a30db3d-9d66-49f9-9ab2-c8d8a8f06caf.png "`Cheat Engine` debugger settings.")

### Retrieve instructions from Cheat Engine into IDA
Finally, we can retrieve instructions from the debugger (`Cheat Engine`) in `IDA`, since the game architecture is `x64`, the base address is `0x140000000`. We need to add the offset retrieve from `Cheat Engine`, for example: `cubeworld.exe+96579`, and then look for the instructions at `0x140096579` in `IDA`.

![Cheat Engine](https://user-images.githubusercontent.com/48086737/234104626-49027dbd-1217-4dc1-9b63-76472451da0f.png "Instructions in the Cheat Engine debugger.")
![IDA Pro](https://user-images.githubusercontent.com/48086737/234104667-7e9ca178-3b5d-4410-85c9-a9581ace2044.png "Instructions retrieved in `IDA`.")

Our game hacking environment is now set up, we are ready to reverse the game and make a cheat !

{{< github repo="adamhlt/Cube-World-Reversing" >}}
