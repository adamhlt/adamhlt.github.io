---
title: "Cube World Reversing - Cheat UI & LocalPlayer"
date: 2022-05-05T17:54:41+02:00
draft: false
tags: [Game Hacking, Reverse Engineering]
summary: "Making the UI for our cheat using ImGui D3D11 and start reverse the game to retrieve the LocalPlayer to start implementing our cheat features."
series: ["Cube World Reversing"]
series_order: 2
---

## Setup the UI
The first thing we need to do is setup a menu for our cheat, the most common technique is to hook DirectX and integrate ImGui to make our menu. We can try this approach since Cube World use DirectX 11. 
After trying differents techniques, the hook works but it brokes the Cube World rendering.

![Cube World ImGui](https://user-images.githubusercontent.com/48086737/234317926-5d3b1c6b-d3a1-4dba-8ce3-42b2267d87c2.png)

As you can see we cannot use this technique to create our menu. 
The second solution is to create a new window with DirectX 11 and integrate ImGui, the menu will be on an external window but the cheat is still internal.
To do that I will use my [project](https://github.com/adamhlt/ImGui-Standalone) and compiled it as an DLL, you can look at the project, everything is setup and you just need to choose if you want to compile it as `DLL` or `EXE`.

{{< github repo="adamhlt/ImGui-Standalone" >}}

![Cube World External](https://user-images.githubusercontent.com/48086737/234318016-42c862fc-acb2-4abf-8ead-718b92078717.png)

Now the UI is setup and our cheat is internal.

## Retrieve the LocalPlayer Address
Now I guess the best thing to do is to retrieve the `LocalPlayer`.
### How to find the LocalPlayer ?
In my opinion the best way to find the `LocalPlayer` is : 
 1. Find the health (Scan then lose life, scan, repeat...).
 2. Check what write to this address with [Cheat Engine](https://www.cheatengine.org)
 3. Then subtract the health address with the health offset we find.

So the first thing we need to do now is scanning for the health, in my case my character has `128hp`, you should be careful the health in Cube World is represented as `float`.

![Scan 1](https://user-images.githubusercontent.com/48086737/234318143-61d79161-e3c3-4377-8f2b-626271d3ba9e.png)

Then I decrease my health with fall damage and I re-scan.

![Scan 2](https://user-images.githubusercontent.com/48086737/234318238-f30aee79-ccee-43d6-94f1-29529b069e90.png)

Next I test the addresses by changing the value and find the good address.

![Scan 3](https://user-images.githubusercontent.com/48086737/234318277-0cd313c4-4778-4fab-b62f-e314a3b6d180.png)

Now I attach the debugger to see what write to this address.

![Scan 4](https://user-images.githubusercontent.com/48086737/234318302-ae3d4add-d37a-46d5-b6be-8a5469b99556.png)

After decrease my health again with fall damage we can see what write to the health address.

![Scan 5](https://user-images.githubusercontent.com/48086737/234318333-ba1ace65-67ba-4297-87a7-a32c8a7afee4.png)

We got what write to the address, as you can see the `R13` register contain the address of the `LocalPlayer` and the offset of the health is `0x180`. I choose this instruction because after some investigation this correspond to the fall damage calculation and health decrease.

![Scan 7](https://user-images.githubusercontent.com/48086737/234318396-c3b94189-008f-4e75-850f-9ef7539fba53.png)

Finally we got the `LocalPlayer` and the health address and offset.

![Scan 8](https://user-images.githubusercontent.com/48086737/234318425-0fde6666-7790-4427-b761-be56a6fd5136.png)

Since the `LocalPlayer` address or health address are not static addresses we need to find a way to retrieve the `LocalPlayer` address at every game start. There are different techniques to do that like pointer scan, hook...

##  Hook the loop
### Trying Pointer Scan
The first approach when you are trying to find a static way to retrieve the `LocalPlayer` is to look at pointer scan.  
Pointer scan is basically brute forcing offsets, I will not explain how it works, this a basic technique and you can find a lot of resources on Google.

To test it I use [Cheat Engine](https://www.cheatengine.org) again. If you don't know how to use pointer scan with Cheat Engine look at [this](https://guidedhacking.com/threads/cheat-engine-how-to-pointer-scan-with-pointermaps.9739/). In my case I had no result with pointer scan, every time I restart the game the formers addresses 

### Look into the game
Previously we found an instruction which modify our health, with this logic the game need to retrieve the LocalPlayer to modify his health. So we can try to find how the game find the LocalPlayer. We start to look at the instruction we found before which is at :

> `cubeworld.exe + 0x2BEC30` in my case `0x1402BEC30`

```
0x1402BEC23     movss   xmm0, dword ptr [r13+180h]
0x1402BEC2C     subss   xmm0, xmm6  
0x1402BEC30     movss   dword ptr [r13+180h], xmm0
```
**Decompilation :**
``` cpp
*(float*)(player + 0x180) = *(float*)(player + 0x180) - fall_damage;
```
Now we just need to find in IDA Pro where the `R13` register / player is initialize in the function. We can see that the `R13` register is set at :
> `cubeworld.exe + 0x2BB969` in my case `0x1402BB969`

```
0x1402BB969     mov   r13, rdx
```
**Decompilation :**
``` cpp
player = param_2
```


Since Cube World is a x64 executable, `RDX` is the second parameter of the function according to [x64 calling convention](https://docs.microsoft.com/fr-fr/cpp/build/x64-calling-convention?view=msvc-170) and it mean that `RDX` is an `INT64`.

So to test if we can retrieve the `LocalPlayer` by hooking this function, we can put a breakpoint at this address and get the value of `RDX`.

![Debug 1](https://user-images.githubusercontent.com/48086737/234318503-75ad246c-c1df-465c-9403-db23aa4f7e8c.png)
![Reclass 1](https://user-images.githubusercontent.com/48086737/234318576-b04393c4-cd58-4709-8545-80247429c9a1.png)

As you can we successfully retrieve the address of the `LocalPlayer`, at the offset `0x180` we found the health as expected.
Unfortunately when we are trying to modify datas like health the datas are not updated and this has no effect. So this address seems to be a "copy" of the `LocalPlayer`.

![Health](https://user-images.githubusercontent.com/48086737/234318629-5fe66b8c-cc75-4af7-be24-f4794b3846de.png)

### Find another way
Even if the first try was not successful it is not a problem, the game need to retrieve the `LocalPlayer` at many places. Fortunately when I was looking for fall damage I found another offset, `LocalPlayer + 0x3C` give the "sum of gravity forces", I explain :

![Gravity](https://user-images.githubusercontent.com/48086737/234318732-236c09dc-83a6-4f8e-9d81-33555aed3591.png)

If the float located at `LocalPlayer + 0x3C` is positive the value of the `Z` axis of the `LocalPlayer` will increase, else if the value is negative the `LocalPlayer` will go down. When your player jump the value is set to `10.0f`.

> `cubeworld.exe + 0x9D443` in my case `0x14009D443`

![Jump 1](https://user-images.githubusercontent.com/48086737/234318870-3d7e24ac-9a2a-4da5-9720-8a866950ffc2.png)
![Jump 2](https://user-images.githubusercontent.com/48086737/234318891-2b4b10ca-04c1-4c3b-9c4c-0db3d93f3b14.png)

The first parameter is in `RDI` because upper in the code `RCX` is moved into `RDI`, else as you can see `LocalPlayer` is set to `0x41200000` which is `10.0f` in hexadecimal. We can easily see how the `LocalPlayer` is retrieved by the game :

```
mov     rax, [rdi+8]
mov     rcx, [rax+448h]
```

As previously we can try to get the `LocalPlayer` with a breakpoint.

![Jump 3](https://user-images.githubusercontent.com/48086737/234318937-467651c5-3ab2-420e-a157-52e4e2db243e.png)
![Jump 4](https://user-images.githubusercontent.com/48086737/234318950-1b409354-102e-4210-b817-5958a5bae6af.png)

As you can see we successfully retrieve the `LocalPlayer` and now we can modify datas. The final step is to hook the function and retrieve the `LocalPlayer` with the first argument.

{{< github repo="adamhlt/Cube-World-Reversing" >}}
