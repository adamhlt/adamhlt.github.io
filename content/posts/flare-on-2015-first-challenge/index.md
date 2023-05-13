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

## Recon of the PE file

Since this is the first challenge in the series, it is likely to be the easiest. Still, it is interesting to see if the executable is packed. To check we will use `Detect It Easy` to see if the software detects a packer and take a look at the `entropy` of the `PE` file data.

![Capture1](https://github.com/adamhlt/adamhlt.github.io/assets/48086737/058f27a3-dbac-4408-8995-e7ee12d695e5)
![Capture2](https://github.com/adamhlt/adamhlt.github.io/assets/48086737/68f6a919-991c-48e4-94d6-63dd7acbcf64)
![Capture3](https://github.com/adamhlt/adamhlt.github.io/assets/48086737/6e24771c-44ca-4106-95e4-b5e95d011567)
![Capture4](https://github.com/adamhlt/adamhlt.github.io/assets/48086737/2ed375ef-ee89-430f-858b-3b6e132f4b85)
![Capture5](https://github.com/adamhlt/adamhlt.github.io/assets/48086737/0a7ef483-2503-4d8b-b050-840595a989df)
![Capture6](https://github.com/adamhlt/adamhlt.github.io/assets/48086737/c37f5bf8-486a-47c3-aef8-6156219eeba9)
![Capture7](https://github.com/adamhlt/adamhlt.github.io/assets/48086737/34507c69-d3d8-4265-a15d-5140206f653a)
![Capture8](https://github.com/adamhlt/adamhlt.github.io/assets/48086737/458426b7-3641-4413-9cdf-662b41cfb230)
