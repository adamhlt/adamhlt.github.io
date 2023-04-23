---
title: "Cube World Reversing - Unpack the game"
date: 2022-02-15T17:54:41+02:00
draft: false
tags: [Game Hacking, Reverse Engineering]
---

{{< lead >}}
I show you how to unpack Cube World, a game packed with Steam DRM, to make a cheat on it.
{{< /lead >}}

{{< alert cardColor="#e63946" iconColor="white" textColor="white">}}
I don't give any information about game cracking, I have a legitimate version of Cube World (v1.0) on Steam and that's why I need to remove Steam DRM to analyze the game.
{{< /alert >}}

## Why Cube World ?

[Cube World](https://store.steampowered.com/app/1128000/Cube_World/) is an action role-playing game developed and published by Picroma. The alpha version was released in 2013, today the official version is available on Steam. I choose this game because there are not alot of informations about it in the game hacking community, you can find some Cheat Engine tables but not more. That's why I choose this game, this force me to do everything by myself and in my opinion it is a great way to train.  

The goal of this series is to reverse the game and to implements some cheat functions like god mode, fly hack, speed hack... Also I will try to make a great C++ structure for the game using [ReClass](https://github.com/ReClassNET/ReClass.NET).

My main goal is to train my reverse engineering skills.

## How to start ?

Before starting reversing the game, we need some tools to do it :
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

If you are trying to load the game into IDA, you will get errors like in the follwing screenshots (I guess it is the same thing into Ghidra).

We also have a weird ".text" section and alot of "nullsub" functions.

![Loading Error](https://cdn.devdojo.com/images/february2022/Capture%201%20(error).png)
![.text Section](https://cdn.devdojo.com/images/february2022/Capture%201%20(text%20section).png)
![Section Zoom](https://cdn.devdojo.com/images/february2022/Capture%201%20(text%20section)%20zoom.png)

We can take a quick look on what happen and try to fix it.

![Sections](https://cdn.devdojo.com/images/february2022/Capture%203.png)
![Entrypoint](https://cdn.devdojo.com/images/february2022/Capture%202.png)
![Entrypoint zoom](https://cdn.devdojo.com/images/february2022/Capture%202%20zoom.png)

As you can see we have two code sections and the entry point is in the ".bind" section, there are alot of weird things and we can deduce that the game is packed with maybe encrypted ".text" section. We need to take a closer look on what is going on with the PE file.

## Analyse the PE file

So to know what is the problem we analyze the PE file with PE Bear and detect it easy.

![PE Bear](https://cdn.devdojo.com/images/february2022/Capture%204.png)

![Steam Stub](https://cdn.devdojo.com/images/february2022/Capture%205.png)

As you can see the entry point is in the ".bind" section, and with Detect I Easy we can see that the game is packed with Steam packer/DRM.
We need to unpack the game the reverse it.

## Dump the game

### Use Steamless
Now we know that the game is packed with Steam DRM, you can find some informations about the DRM [here](https://www.pcgamingwiki.com/wiki/User:Cyanic/Steam_DRM). The next step is to unpack the game, it is useless to do it manually since [Steamless](https://github.com/atom0s/Steamless) works great.
If you want to try unpack the game manually or want to know how Steam DRM works you can at [Steamless](https://github.com/atom0s/Steamless) Github, every step of unpacking are described in the code.

![Steamless](https://cdn.devdojo.com/images/february2022/Capture%206.png)

![Steamless Unpack](https://cdn.devdojo.com/images/february2022/Capture%207.png)

### Analyse unpacked file
As you can see it is very easy to unpack the game with [Steamless](https://github.com/atom0s/Steamless), if you try to launch it, it will crash, the new version of Cube World is very dependant of Steam API and without patching functions the game will not be functional. In our case it is not important, since the DRM unpack the game in memory we can debug the game and use IDA with the upacked game file.

![New Sections](https://cdn.devdojo.com/images/february2022/Capture%208.png)
![New .text Section](https://cdn.devdojo.com/images/february2022/Capture%209.png)
![New Section Zoom](https://cdn.devdojo.com/images/february2022/Capture%209%20zoom.png)

Presently every sections of the DRM have been removed, the `.text` section looks great, "nullsub" functions are gone and the analysis is way faster. The next step is to check if we can retrieve instructions from the debugger into IDA.

## Debugging
### Use VEH debugger
The last step is to setup our environment, if you are trying to use Cheat Engine with Windows debugger, Cube World will crash in few cases, so I recommend to use VEH debugger with Cheat Engine (I never faced problem with VEH, but this is only for Cube World). In the follwing screenshots I show you how to configure Cheat Engine and what happen when you are trying to use Windows debugger.

![Crash](https://github.com/adamhlt/Reversing-Cube-World/blob/main/Ressouces/crash.gif?raw=true)
![VEH](https://cdn.devdojo.com/images/february2022/Capture%2011.png)

### Retrieve instructions
Finaly we can retrieve instructions from the debugger into IDA, since our game is x64 in IDA the base address is `0x140000000` so we just need to add the offset from Cheat Engine, so `cubeworld.exe+96579` is `0x140096579` in IDA.

![Cheat Engine](https://cdn.devdojo.com/images/february2022/Capture%2012.png)
![IDA Pro](https://cdn.devdojo.com/images/february2022/Capture%2013.png)

Our game hacking environment is now setup, we are ready to reverse the game and make a cheat !

{{< github repo="adamhlt/Cube-World-Reversing" >}}
