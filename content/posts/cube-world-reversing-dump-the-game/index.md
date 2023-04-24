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

[Cube World](https://store.steampowered.com/app/1128000/Cube_World/) is an action role-playing game developed and published by Picroma. The alpha version was released in 2013, today the official version is available on Steam. I choose this game because there are not a lot of information about it in the game hacking community, you can find some Cheat Engine tables but not more. That's why I choose this game, this force me to do everything by myself and in my opinion it is a great way to train.  

The goal of this series is to reverse the game and to implement some cheat functions like god mode, fly hack, speed hack...
The game Cube World use his own game engine, which is also called Cube World, that's why it can be a great reverse enginnering challenge to reverse it.

## How to start ?

Before starting to reverse the game, we need some tools to do it :
- [Cheat Engine](https://www.cheatengine.org)
- [x64dbg](https://x64dbg.com)
- [ReClass.NET](https://github.com/ReClassNET/ReClass.NET)
- [IDA Pro](https://hex-rays.com/ida-pro/) / [Ghidra](https://ghidra-sre.org)
- [PE Bear](https://github.com/hasherezade/pe-bear-releases)
- [Detect It Easy](https://github.com/horsicq/Detect-It-Easy)

Some of these tools will be used only in this part of the reversing.
The first thing we want to do is to load the game in IDA (or Ghidra).

```console
[steam_path]\steamapps\common\Cube World\cubeworld.exe
```

If you are trying to load the game into IDA, you will get errors like in the following screenshots (I guess it is the same thing into Ghidra).

We also have a weird `.text` section and a lot of `nullsub` functions.

![Loading Error](https://user-images.githubusercontent.com/48086737/234102864-a7f784dd-46b5-4a8c-a646-d28aa2c68ecf.png)
![.text Section](https://user-images.githubusercontent.com/48086737/234102919-2e8612bd-1265-40b6-985c-0ad899b17f7f.png)
![Section Zoom](https://user-images.githubusercontent.com/48086737/234102903-1a1c7bfc-ec87-4fb4-a502-9c78dabf35c7.png)

We can take a quick look on what happen and try to fix it.

![Sections](https://user-images.githubusercontent.com/48086737/234103254-6dfc50b4-801c-4890-b2ff-0b88d622d6a0.png)
![Entrypoint](https://user-images.githubusercontent.com/48086737/234103268-b8d50091-022e-4c40-b5a7-51cb34db2e9c.png)
![Entrypoint zoom](https://user-images.githubusercontent.com/48086737/234103277-2aae231d-3409-4349-b733-c731a60784b4.png)

As you can see we have two code sections and the entry point is in the `.bind` section, there are a lot of weird things and we can deduce that the game is packed with maybe encrypted `.text` section. We need to take a closer look on what is going on with the PE file.

## Analyse the PE file

So to know what is the problem we analyze the PE file with PE Bear and detect it easy.

![PE Bear](https://user-images.githubusercontent.com/48086737/234103386-3755cb8b-0ed4-4579-9cc2-dbeb3896d1f7.png)

![Steam Stub](https://user-images.githubusercontent.com/48086737/234103414-5baef205-7c17-4e98-b0f3-8f13e9f7bc23.png)

As you can see the entry point is in the `.bind` section, and with Detect I Easy we can see that the game is packed with Steam packer/DRM.
We need to unpack the game the reverse it.

## Dump the game

### Use Steamless
Now we know that the game is packed with Steam DRM, you can find some information about the DRM [here](https://www.pcgamingwiki.com/wiki/User:Cyanic/Steam_DRM). The next step is to unpack the game, it is useless to do it manually since [Steamless](https://github.com/atom0s/Steamless) works great.
If you want to try unpack the game manually or want to know how Steam DRM works you can at [Steamless](https://github.com/atom0s/Steamless) Github, every step of unpacking is described in the code.

![Steamless](https://user-images.githubusercontent.com/48086737/234103966-91ce7628-642e-4474-ab48-a84742876b89.png)

![Steamless Unpack](https://user-images.githubusercontent.com/48086737/234103987-5abf6a54-7e42-4b00-a498-6dd528eb957a.png)

### Analyse unpacked file
As you can see it is very easy to unpack the game with [Steamless](https://github.com/atom0s/Steamless), if you try to launch it, it will crash, the new version of Cube World is very dependant of Steam API and without patching functions the game will not be functional. In our case it is not important, since the DRM unpacks the game in memory we can debug the game and use IDA with the unpacked game file.

![New Sections](https://user-images.githubusercontent.com/48086737/234104204-079f8559-aebd-4cb3-829d-52f77f7f30ee.png)
![New .text Section](https://user-images.githubusercontent.com/48086737/234104226-86df20b1-c2f9-4b22-b248-de7fbe99792e.png)
![New Section Zoom](https://user-images.githubusercontent.com/48086737/234104237-bb8c0c69-05a8-42e1-94c2-d33826debef3.png)

Presently, every section of the DRM has been removed, the `.text` section looks great, `nullsub` functions are gone and the analysis is way faster. The next step is to check if we can retrieve instructions from the debugger into IDA.

## Debugging
### Use VEH debugger
The last step is to setup our environment, if you are trying to use Cheat Engine with Windows debugger, Cube World will crash in a few cases, so I recommend to use VEH debugger with Cheat Engine (I never faced a problem with VEH, but this is only for Cube World). In the following screenshots I show you how to configure Cheat Engine and what happen when you are trying to use Windows debugger.

![VEH](https://user-images.githubusercontent.com/48086737/234104437-9a30db3d-9d66-49f9-9ab2-c8d8a8f06caf.png)

### Retrieve instructions
Finally we can retrieve instructions from the debugger into IDA, since our game is x64 in IDA, the base address is `0x140000000` so we just need to add the offset from Cheat Engine, so `cubeworld.exe+96579` is `0x140096579` in IDA.

![Cheat Engine](https://user-images.githubusercontent.com/48086737/234104626-49027dbd-1217-4dc1-9b63-76472451da0f.png)
![IDA Pro](https://user-images.githubusercontent.com/48086737/234104667-7e9ca178-3b5d-4410-85c9-a9581ace2044.png)

Our game hacking environment is now setup, we are ready to reverse the game and make a cheat !

{{< github repo="adamhlt/Cube-World-Reversing" >}}
