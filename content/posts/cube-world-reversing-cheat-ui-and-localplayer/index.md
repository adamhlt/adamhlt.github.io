---
title: "Cube World Reversing - Cheat UI & LocalPlayer"
date: 2022-05-05T17:54:41+02:00
draft: false
tags: [Game Hacking, Reverse Engineering]
summary: "Making the UI for our cheat using ImGui D3D11 and start reverse the game to retrieve the LocalPlayer to start implementing our cheat features."
series: ["Cube World Reversing"]
series_order: 2
---

## Setting up the Cheat UI
The first thing we need to do is set up a menu for our cheat, the most common technique is to hook `DirectX` and integrate `ImGui` to make our menu. We can try this approach as Cube World uses `DirectX 11`.

After trying different techniques, the hook works but it breaks the Cube World rendering.

![Cube World ImGui](https://user-images.githubusercontent.com/48086737/234317926-5d3b1c6b-d3a1-4dba-8ce3-42b2267d87c2.png "Cube World rendering breaks when `ImGui` menu is injected.")

As you can see, we cannot use this technique to create our menu. 
The second solution is to create a new window with `DirectX 11` and integrate `ImGui`, the menu will be on an external window but the cheat will still be internal.

To do this I will use my project and compile it as `DLL`, you can look at the project, everything is set up and you just have to choose if you want to compile it as `DLL` or `EXE`.

{{< github repo="adamhlt/ImGui-Standalone" >}}

![Cube World External](https://user-images.githubusercontent.com/48086737/234318016-42c862fc-acb2-4abf-8ead-718b92078717.png "Using external windows we keep the cheat internal and the rendering is fine.")

Now the UI is set up and our cheat is internal, so it will be developed as a `DLL`.

## Finding out the address of the LocalPlayer
The best thing to do now is to get the `LocalPlayer`.

In my opinion the best way to find the `LocalPlayer` address is to : 
 1. Find the health (scan then lose life, scan, repeat...).
 2. Check which instruction writes to this address with [Cheat Engine](https://www.cheatengine.org).
 3. Then subtract the health address with the health offset we found.

So the first thing we need to do now is scan for health. In my case my character has `128hp`, you should note that health in Cube World is a `float`.

![Scan 1](https://user-images.githubusercontent.com/48086737/234318143-61d79161-e3c3-4377-8f2b-626271d3ba9e.png "First scan for health in `Cheat Engine`.")

Then I take some fall damage to lower my health. Then I rescan my health with the new value and remove false addresses.

![Scan 2](https://user-images.githubusercontent.com/48086737/234318238-f30aee79-ccee-43d6-94f1-29529b069e90.png "A few addresses remain after several scans.")

Next, when there are only a few addresses left, I test the addresses by changing or blocking their values. To find the good address, I looked at the change in value.

![Scan 3](https://user-images.githubusercontent.com/48086737/234318277-0cd313c4-4778-4fab-b62f-e314a3b6d180.png "I kept the address which is valid and filled the life bar of my character.")

Now, to see which instruction writes to the health address, I attach the Cheat Engine debugger.

![Scan 4](https://user-images.githubusercontent.com/48086737/234318302-ae3d4add-d37a-46d5-b6be-8a5469b99556.png "The debugger is attached, we just need to change the health value to see instructions.")

After decreasing my health again with fall damage we can see the instructions that write to the health address.

![Scan 5](https://user-images.githubusercontent.com/48086737/234318333-ba1ace65-67ba-4297-87a7-a32c8a7afee4.png "Few instructions write to health address.")

We have the instruction that writes to the address, as you can see the `R13` register contains the address of the `LocalPlayer` and the health offset is `0x180`. I chose this instruction because after some investigation this corresponds to the fall damage calculation and the health decrease, a good clue is the `count`, since I only take fall damage once, the third instruction is the best candidate.

![Scan 7](https://user-images.githubusercontent.com/48086737/234318396-c3b94189-008f-4e75-850f-9ef7539fba53.png "Details of the instruction.")

Finally we got the `LocalPlayer` and the health address and offset.

![Scan 8](https://user-images.githubusercontent.com/48086737/234318425-0fde6666-7790-4427-b761-be56a6fd5136.png "Adress of the `LocalPlayer` in `Cheat Engine`.")

Since the `LocalPlayer` address or the health address are not static addresses, we need to find a way to retrieve the `LocalPlayer` address each time the game starts. There are several techniques to do this, such as pointer scan, hook...

##  Hook the game loop
### Trying pointer scan
The first approach when trying to find a static address is to retrieve the `LocalPlayer` and then do a pointer scan after restarting the game.

Pointer scan is basically brute forcing offsets, I will not explain how it works, it is a basic technique and you can find lots of resources on Google.

If you don't know how to do a pointer scan with `Cheat Engine` look at [this](https://guidedhacking.com/threads/cheat-engine-how-to-pointer-scan-with-pointermaps.9739/). In my case I had no result with the pointer scan, every time I restarted the game the formers addresses were invalid.

### Look into the game
Previously we found an instruction that modifies our health, with this logic the game must retrieve the `LocalPlayer` to modify his health. So we can try to find out how the game finds the `LocalPlayer`. We can start by looking at the instruction we found before, which is at :

> `cubeworld.exe + 0x2BEC30` in `Cheat Engine` and `0x1402BEC30` in `IDA`

```
0x1402BEC23     movss   xmm0, dword ptr [r13+180h]
0x1402BEC2C     subss   xmm0, xmm6  
0x1402BEC30     movss   dword ptr [r13+180h], xmm0
```
**Decompilation :**
{{< highlight cpp >}}
*(float*)(player + 0x180) = *(float*)(player + 0x180) - fall_damage;
{{< / highlight >}}

To retrieve the address of the `LocalPlayer` we need to find out in `IDA` where the `R13` register is set in the function. We can see that the `R13` register is set at :
> `cubeworld.exe + 0x2BB969` in `Cheat Engine` and `0x1402BB969` in `IDA`

```
0x1402BB969     mov   r13, rdx
```
**Decompilation :**
``` cpp
player = param_2
```

Since the game architecture is `x64`, this means that `RDX` is the second parameter of the function according to the [x64 calling convention](https://docs.microsoft.com/fr-fr/cpp/build/x64-calling-convention?view=msvc-170) and it implies that `RDX` is an `INT64`.

To confirm that we have successfully retrieved the `LocalPlayer` by hooking this function, we can set a breakpoint at this address and look at the value of `RDX`.

![Debug 1](https://user-images.githubusercontent.com/48086737/234318503-75ad246c-c1df-465c-9403-db23aa4f7e8c.png "Breakpoint hit in the `Cheat Engine` debugger.")
![Reclass 1](https://user-images.githubusercontent.com/48086737/234318576-b04393c4-cd58-4709-8545-80247429c9a1.png "Analysis of the retrieved address in `ReClass.NET`.")

As we have successfully retrieved the address of the `LocalPlayer`, we should find the character's health at offset `0x180`, which is the case here.

Unfortunately, when we try to change the health of the character using `Cheat Engine`, the health is not updated and this has no effect. So this address seems to be a "copy" of the `LocalPlayer`.

![Health](https://user-images.githubusercontent.com/48086737/234318629-5fe66b8c-cc75-4af7-be24-f4794b3846de.png "Trying to change the value of the character's health, using `Cheat Engine`.")

### Find another way
Even if the first attempt was not successful, it is not a problem, the game needs to retrieve the `LocalPlayer` in many places. Fortunately, when I was looking for fall damage, I found another offset, `LocalPlayer + 0x3C`, which gives the "sum of gravitational forces", I explain:

![Gravity](https://user-images.githubusercontent.com/48086737/234318732-236c09dc-83a6-4f8e-9d81-33555aed3591.png "Scheme representing the sum of the gravitational forces.")

If the `float` at `LocalPlayer + 0x3C` is positive, the `Z axis` of the `LocalPlayer` will increase, otherwise if the value is negative, the `LocalPlayer` will decrease. When your player jumps, the value is set to `10.0f`. This is the code that makes your character jump by changing the value of the `Z axis`.

> `cubeworld.exe + 0x9D443` in `Cheat Engine` and `0x14009D443` in `IDA`

![Jump 1](https://user-images.githubusercontent.com/48086737/234318870-3d7e24ac-9a2a-4da5-9720-8a866950ffc2.png "Instructions that make your character jump.")

**Decompilation :**
``` cpp
*(DWORD*)(*(QWORD*)(*(QWORD*)(first_parameter + 0x8) + 0x448) + 0x3C) = 10.0f;
```

With the code above we can easily analyse how the game retrieves the `LocalPlayer` and then puts it into the `RCX` register. Then make the character jump by setting the value at `RCX + 3C` to `0x41200000` which is `10.0f` in hexadecimal.

Here is the code to retrieve the `LocalPlayer` :

```
mov     rax, [rdi+8]
mov     rcx, [rax+448h]
```

**Decompilation :**
``` cpp
QWORD LocalPlayer = *(QWORD*)(*(QWORD*)(first_parameter + 0x8) + 0x448);
```

As before we can try to get the `LocalPlayer` using a breakpoint in the `Cheat Engine` debugger.

![Jump 3](https://user-images.githubusercontent.com/48086737/234318937-467651c5-3ab2-420e-a157-52e4e2db243e.png "Breakpoint hit in the `Cheat Engine` debugger.")
![Jump 4](https://user-images.githubusercontent.com/48086737/234318950-1b409354-102e-4210-b817-5958a5bae6af.png "`LocalPlayer` view in `ReClass.NET`.")

As you can see, we have successfully retrieved the `LocalPlayer` and can now modify the health or gravity value. The last step is to hook the function and get the `LocalPlayer` using the first argument of the function we found earlier.

{{< github repo="adamhlt/Cube-World-Reversing" >}}
