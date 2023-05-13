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

![Detect It Easy Analysis](https://user-images.githubusercontent.com/48086737/236915927-94ce6530-1fe3-40fa-b64f-ba92eaeb943b.png "The `Detect It Easy` analysis do not return a packer.")

![Detect It Easy Analysis 2](https://user-images.githubusercontent.com/48086737/236916390-0e9963ae-8abb-458e-a08e-32f843b06ef7.png "The analysis show a low entropy.")
