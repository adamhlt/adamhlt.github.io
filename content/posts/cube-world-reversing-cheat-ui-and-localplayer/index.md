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
![Cube World ImGui.png](https://cdn.devdojo.com/images/may2022/cube-world-imgui1.png)
As you can see we cannot use this technique to create our menu. 
The second solution is to create a new window with DirectX 11 and integrate ImGui, the menu will be on an external window but the cheat is still internal.
To do that I will use my [project](https://github.com/adamhlt/ImGui-Standalone) and compiled it as an DLL, you can look at the project, everything is setup and you just need to choose if you want to compile it as `DLL` or `EXE`.

{{< github repo="adamhlt/ImGui-Standalone" >}}

![Cube World External.png](https://cdn.devdojo.com/images/may2022/Cube%20World%20External.png)

Now the UI is setup and our cheat is internal.

## Retrieve the LocalPlayer Address
Now I guess the best thing to do is to retrieve the `LocalPlayer`.
### How to find the LocalPlayer ?
In my opinion the best way to find the `LocalPlayer` is : 
 1. Find the health (Scan then lose life, scan, repeat...).
 2. Check what write to this address with [Cheat Engine](https://www.cheatengine.org)
 3. Then subtract the health address with the health offset we find.

So the first thing we need to do now is scanning for the health, in my case my character has `128hp`, you should be careful the health in Cube World is represented as `float`.
![Scan 1.png](https://cdn.devdojo.com/images/may2022/Scan%201.png)

Then I decrease my health with fall damage and I re-scan.
![Scan 2.png](https://cdn.devdojo.com/images/may2022/Scan%202.png)

Next I test the addresses by changing the value and find the good address.
![Scan 3.png](https://cdn.devdojo.com/images/may2022/Scan%203.png)

Now I attach the debugger to see what write to this address.
![Scan 4.png](https://cdn.devdojo.com/images/may2022/Scan%204.png)

After decrease my health again with fall damage we can see what write to the health address.
![Scan 5.png](https://cdn.devdojo.com/images/may2022/Scan%205.png)

We got what write to the address, as you can see the `R13` register contain the address of the `LocalPlayer` and the offset of the health is `0x180`. I choose this instruction because after some investigation this correspond to the fall damage calculation and health decrease.
![Scan 7.png](https://cdn.devdojo.com/images/may2022/Scan%207.png)

Finally we got the `LocalPlayer` and the health address and offset.
![Scan 8.png](https://cdn.devdojo.com/images/may2022/Scan%208.png)

Since the `LocalPlayer` address or health address are not static addresses we need to find a way to retrieve the `LocalPlayer` address at every game start. There are different techniques to do that like pointer scan, hook...

##  Hook the loop
### Trying Pointer Scan
The first approach when you are trying to find a static way to retrieve the `LocalPlayer` is to look at pointer scan.  
Pointer scan is basically brute forcing offsets, I will not explain how it works, this a basic technique and you can find a lot of resources on Google.

To test it I use [Cheat Engine](https://www.cheatengine.org) again. If you don't know how to use pointer scan with Cheat Engine look at [this](https://guidedhacking.com/threads/cheat-engine-how-to-pointer-scan-with-pointermaps.9739/). In my case I had no result with pointer scan, every time I restart the game the formers addresses 

### Look into the game
Previously we found an instruction which modify our health, with this logic the game need to retrieve the LocalPlayer to modify his health. So we can try to find how the game find the LocalPlayer. We start to look at the instruction we found before which is at :

> `cubeworld.exe + 0x2BEC30` in my case `0x1402BEC30`

``` console
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

``` console
0x1402BB969     mov   r13, rdx
```
**Decompilation :**
``` cpp
player = param_2
```


Since Cube World is a x64 executable, `RDX` is the second parameter of the function according to [x64 calling convention](https://docs.microsoft.com/fr-fr/cpp/build/x64-calling-convention?view=msvc-170) and it mean that `RDX` is an `INT64`.

So to test if we can retrieve the `LocalPlayer` by hooking this function, we can put a breakpoint at this address and get the value of `RDX`.

![Debug 1.png](https://cdn.devdojo.com/images/may2022/Debug%201.png)
![Reclass 1.png](https://cdn.devdojo.com/images/may2022/Reclass%201.png)

As you can we successfully retrieve the address of the `LocalPlayer`, at the offset `0x180` we found the health as expected.
Unfortunately when we are trying to modify datas like health the datas are not updated and this has no effect. So this address seems to be a "copy" of the `LocalPlayer`.

![Health.png](https://cdn.devdojo.com/images/may2022/Health.png)

### Find another way
Even if the first try was not successful it is not a problem, the game need to retrieve the `LocalPlayer` at many places. Fortunately when I was looking for fall damage I found another offset, `LocalPlayer + 0x3C` give the "sum of gravity forces", I explain :

![Gravity.png](https://cdn.devdojo.com/images/may2022/gravity-21.png)

If the float located at `LocalPlayer + 0x3C` is positive the value of the `Z` axis of the `LocalPlayer` will increase, else if the value is negative the `LocalPlayer` will go down. When your player jump the value is set to `10.0f`.

> `cubeworld.exe + 0x9D443` in my case `0x14009D443`

![Jump 1.png](https://cdn.devdojo.com/images/may2022/Jump%201.png)
![Jump 2.png](https://cdn.devdojo.com/images/may2022/Jump%202.png)

The first parameter is in `RDI` because upper in the code `RCX` is moved into `RDI`, else as you can see `LocalPlayer` is set to `0x41200000` which is `10.0f` in hexadecimal. We can easily see how the `LocalPlayer` is retrieved by the game :

``` console
mov     rax, [rdi+8]
mov     rcx, [rax+448h]
```

As previously we can try to get the `LocalPlayer` with a breakpoint.

![Jump 3.png](https://cdn.devdojo.com/images/may2022/Jump%203.png)
![Jump 4.png](https://cdn.devdojo.com/images/may2022/Jump%204.png)

As you can see we successfully retrieve the `LocalPlayer` and now we can modify datas. The final step is to hook the function and retrieve the `LocalPlayer` with the first argument.

{{< github repo="adamhlt/Cube-World-Reversing" >}}