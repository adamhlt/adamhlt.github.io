---
title: "LiteX - Add JTAG PHY interface to VexRiscV CPU"
date: 2023-02-23T17:54:41+02:00
draft: true
tags: [Hardware, LiteX, FPGA]
summary: "How to add a JTAG PHY interface to VexRiscV CPU into LiteX SoC generator and debug the CPU with OpenOCD and GDB."
---

## Introduction

In this tutorial, I will demonstrate how to add a JTAG interface to a VexRiscv CPU and integrate it in LiteX SoC generator. By adding a physical JTAG interface to the VexRiscv CPU, we can easily debug and test our design using standard JTAG tools. 

It is important to note that the JTAG interface requires additional pins to be added to the FPGA. Therefore, we need to modify the platform file of the LiteX SoC generator to include these additional pins. Once this is done, we can then add the JTAG module to the VexRiscv CPU and connect it to the additional pins. Finally, we can regenerate the SoC and program it onto our FPGA board to test the JTAG interface.

## Prerequisites

Before starting make sure you have all you need to follow the tutorial :

- A working installation of LiteX
- An On-chip Debugger
- A JTAG programmer
- A FPGA board

### My setup

- **OS** : Ubuntu 20.04
- **LiteX** : [Tag 2022.08](https://github.com/enjoy-digital/litex/tree/2022.08)
- **Toolchain** : riscv64-unknown-elf-gcc
- **On-chip Debugger** : [OpenOCD](https://github.com/SpinalHDL/openocd_riscv) for VexRiscV
- **Board** : Digilent Basys3

### Install

**riscv64-unknown-elf-gcc toolchain**

```
sudo apt install gcc-riscv64-unknown-elf
```

**OpenOCD for VexRiscV**

If you have any errors when installing OpenOCD go check out the Github [here](https://github.com/SpinalHDL/openocd_riscv).

```
git clone https://github.com/SpinalHDL/openocd_riscv.git
cd openocd_riscv
sudo apt-get install libtool automake libusb-1.0.0-dev texinfo libusb-dev libyaml-dev pkg-config
./bootstrap
./configure --enable-ftdi --enable-dummy
make
sudo make install
```

You should now have everything you need to follow the tutorial.

## Resources

{{< github repo="SpinalHDL/openocd_riscv" >}}
<br>
{{< github repo="enjoy-digital/litex" >}}