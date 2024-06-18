---
title: "Damn Vuln IoT SoC - SSTIC 2024"
date: 2024-06-18T17:54:41+02:00
draft: false
tags: [Hardware, LiteX, FPGA, SSTIC]
summary: "Damn Vuln IoT SoC is a modular platform to generate SoC with hardware vulnerabilities, based on an instrumented version of LiteX SoC generator."
---

{{< alert cardColor="#6a89cc" icon="circle-info" iconColor="white" textColor="white">}}
Find all the information and slides on the official SSTIC website [here](https://www.sstic.org/2024/presentation/damn_vuln_iot_soc/).
{{< /alert >}}

## What is Damn Vuln IoT SoC ?

Damn Vuln IoT SoC is a modular platform to generate SoC with hardware vulnerabilities, based on an instrumented version of [LiteX](https://github.com/enjoy-digital/litex) SoC generator.
This tool can be used for educational purposes, CTF or HDL analysis tool.

## What is the goal of the project ?

Attention to securing hardware designs is intensifying due to the ubiquity of computing and communication elements in our daily lives. The continuing unveiling of previously unknown vulnerabilities, exemplified by vulnerabilities such as Spectre and Meltdown, further underlines the growing importance of enhanced security measures. Exploiting such hardware bugs presents significant security challenges, as rectifying them often necessitates modifications to the affected systems' hardware. Therefore, it becomes crucial to comprehend the implications of these vulnerabilities and how to detect them. Addressing this concern, the Damn Vuln IoT SoC tool provides a solution, a modular platform facilitating the seamless integration of hardware description errors into a System on Chip for FPGA. It serves as a valuable resource for understanding and navigating the identification and exploitation of such vulnerabilities. 

## Video presentation

{{<video src="https://static.sstic.org/videos2024/1080p/damn_vuln_iot_soc.mp4" span="6" autoplay="false" muted="false" loop="false">}}

## Ressources

{{< github repo="Damn-Vuln-IoT-SoC/Damn-Vuln-IoT-SoC" >}}
