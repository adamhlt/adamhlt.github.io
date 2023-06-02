---
title: "Flare On 2015 - 0x1 First challenge writeup"
date: 2023-05-08T16:20:41+02:00
draft: True
tags: [Reverse Engineering, Flare On]
summary: "This is the walkthrough of the first challenge in the Flare On 2015 series, how to solve the challenge using IDA Pro and Python."
---

## Introduction

The `Flare-On` Challenge is an annual reverse engineering and hacking challenge created by the cybersecurity company `FireEye`. It is designed to test the skills and knowledge of participants in various areas of cybersecurity, such as malware analysis, cryptography, network analysis, and exploit development. The challenge typically consists of a series of increasingly difficult levels, with each level presenting a unique set of challenges that must be solved in order to progress to the next level.

I will present my analysis and solution for the first challenge of the 2015 `Flare-On` challenge. I advise you to use the article if you are stuck or if you have already done the challenge, otherwise do it by yourself, it is the most formative.

## Look at the challenge

Before we start solving the challenge we can look at what it looks like and what we have to do.

![Challenge Execution](https://github.com/adamhlt/adamhlt.github.io/assets/48086737/34507c69-d3d8-4265-a15d-5140206f653a "Execution of the challenge in the console.")

As you can see the goal of the challenge is to find the valid password. This already gives us an idea of what we will have to look for during the reverse engineering.

## Recon of the PE file

Since this is the first challenge in the series, it is likely to be the easiest. Still, it is interesting to see if the executable is packed. To check we will use `Detect It Easy` to see if the software detects a packer and take a look at the `entropy` of the `PE` file data.

![Detect It Easy Analysis](https://github.com/adamhlt/adamhlt.github.io/assets/48086737/0a7ef483-2503-4d8b-b050-840595a989df "The `Detect It Easy` analysis tells us that it is a 32 bit PE file.")

![Detect It Easy Entropy](https://github.com/adamhlt/adamhlt.github.io/assets/48086737/c37f5bf8-486a-47c3-aef8-6156219eeba9 "We can see that the `entropy` of the data in the sections is low which shows that the file is not packed.")

`Detect It Easy` tells us that the challenge is a `PE32` file, which allows us to choose the right version of `IDA`, and we learn that the file is not packed (which seems normal, we are only at the first challenge) by analyzing the `entropy`.

## Code Analysis

Now that we know that the file is not packaged and that we know its architecture we can start to analyse the challenge code. So first I load the `PE` file into `IDA Pro`.

![IDA Pro 1](https://github.com/adamhlt/adamhlt.github.io/assets/48086737/54fbd4ab-ff03-4d80-b168-c7509765ade5 "Overview of the challenge file in `IDA Pro`.")

![IDA Pro 2](https://github.com/adamhlt/adamhlt.github.io/assets/48086737/058f27a3-dbac-4408-8995-e7ee12d695e5 "Main part of the challenge code.")

As you can see, there is very little code, only one function exists. The code can be split into 3 parts: first, requesting the password and retrieving the password entered by the user, then checking the validity of the password and finally displaying the error or success message. We will first focus on the recovery of the password and then reverse engineer the password check to find the valid password.

### Password retrieval and display

In this first part we will analyse how the program retrieves the password and how it displays messages in the console. Below, you can see the part that comes to ask for the password and retrieves it.

You can see that the program uses the functions `GetStdHandle`, `WriteFile` and `ReadFile`, I will detail their use and with which parameters they are called.

![Display WINAPI](https://github.com/adamhlt/adamhlt.github.io/assets/48086737/68f6a919-991c-48e4-94d6-63dd7acbcf64 "The part of the code that manages the password request and retrieval.")

The first function to be called is the `GetStdHandle` function. This function is part of `WINAPI` and retrieves a handle to the specified standard device (standard input, standard output or standard error). You can find the documentation for the function [here]("https://learn.microsoft.com/en-us/windows/console/getstdhandle").

This function has only one parameter and this parameter can only have 3 values :

* `((DWORD)-10)` (`0xFFFFFFF6` in `UINT32`) : this is the value corresponding to the standard console input and retrieves the text entered by the user. 

* `((DWORD)-11)` (`0xFFFFFFF5` in `UINT32`) : this is the value corresponding to the standard console output and is used to display text in the console.

* `((DWORD)-12)` (`0xFFFFFFF4` in `UINT32`) : this is the value corresponding to the standard console error and is used to display error in the console.

The `HANDLE` retrieved after calling the `GetStdHandle` function will be used to write a message in the console or to retrieve the text entered by the user.

To be able to write to the console, you need to call the `GetStdHandle` function with the `(DWORD)-11` parameter to retrieve the `HANDLE` corresponding to the console output. Then use the `WriteFile` function to write to the terminal output buffer. We can therefore deduce that `[ebp+hFile]` corresponds to the `HANDLE` of the console output.

What we are interested in is how the password is retrieved. To retrieve the text entered by the user in the console, we need to call the `GetStdHandle` function with parameter `(DWORD)-10`. The `HANDLE` returned by the function corresponds to the standard console input; to retrieve the contents of the standard input buffer, we use the `ReadFile` function. A quick analysis of the parameters passed to the `ReadFile` function reveals the number of characters read by the `ReadFile` function, i.e. `0x32`, which is equal to `50` in decimal. We can also see the `HANDLE` passed to the function, which is stored at `[ebp+var_c]`, and finally, what interests us most, the buffer containing the password that has just been read is at address `0x402158`.

### Password verification

Now that we've analysed how the program retrieves the password and where the password is stored, we can move on to analysing password verification.

![Password Verification Overview](https://github.com/adamhlt/adamhlt.github.io/assets/48086737/6e24771c-44ca-4106-95e4-b5e95d011567 "Overview of the password verification assembly code.")

As you can see, the code used to check the password is relatively short.  First, `ecx` is set to `0` with the instruction `xor ecx, ecx`, which may seem to indicate the use of a loop. Next, the password character entered by the user at the index of `ecx` is loaded into the `al` register, then xored with the value `0x7D` and finally compared with a character at the index of `ecx` present in a buffer at address `0x40107B`.  

![Correct Password Buffer](https://github.com/adamhlt/adamhlt.github.io/assets/48086737/2ed375ef-ee89-430f-858b-3b6e132f4b85 "Data in the buffer containing the correct password.")

It would therefore appear that the correct password is in the buffer at address `0x40107B`, but this has been xored, so the operation will have to be reversed to recover the correct password.

Finally, in the second part of the password check, we can see that if the comparison is correct, `ecx` is incremented and compared with the value `0x18` (`24`), which seems to correspond to the size of the password.

## Password recovery

Now we know that the xored password is in the buffer at address `0x40107B` and that it was xored with the key `0x7D`. With all this information we can create a small Python script that will allow us to recover the password in clear text. Since the reverse operation of the `XOR` is the `XOR` itself, we simply need to xor the characters in the buffer with the key `0x7D` to recover the password in clear.

```python
def xor_data(data, key):
    result = []
    for byte in data:
        result.append(byte ^ key)
    return result

def bytes_to_string(data):
    return ''.join(chr(byte) for byte in data)

data = [0x1F, 0x8, 0x13, 0x13, 0x4, 0x22, 0x0E, 0x11, 0x4D, 0x0D, 0x18, 0x3D, 0x1B, 0x11, 0x1C, 0x0F, 0x18, 0x50, 0x12, 0x13, 0x53, 0x1E, 0x12, 0x10]

key = 0x7D

result = xor_data(data, key)
result_string = bytes_to_string(result)

print(result_string)
```

We were able to recover the correct password and successfully complete the challenge.