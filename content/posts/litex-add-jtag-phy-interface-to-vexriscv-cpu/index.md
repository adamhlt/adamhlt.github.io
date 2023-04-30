---
title: "LiteX - Add JTAG PHY interface to VexRiscV CPU"
date: 2023-02-23T17:54:41+02:00
draft: true
tags: [Hardware, LiteX, FPGA]
summary: "How to add a JTAG PHY interface to a VexRiscV CPU in the LiteX SoC generator and debug the CPU with OpenOCD and GDB."
---

## Introduction

In this tutorial I will show how to add a `JTAG` interface to a `VexRiscv` CPU and integrate it into the `LiteX SoC Generator`. By adding a physical `JTAG` interface to the `VexRiscv CPU`, we can easily debug and test our design using standard `JTAG` tools.

It is important to note that the `JTAG` interface requires additional pins to be added to the `FPGA`. Therefore, we need to modify the `platform` file of the `LiteX SoC Generator` to include these additional pins. Once this is done, we can add the `JTAG` module to the `VexRiscv CPU` and connect it to the additional pins. Finally, we can regenerate the SoC and program it on our `FPGA` board to test the `JTAG` interface.

## Prerequisites

Before starting, make sure you have everything you need to follow the tutorial:

- A working installation of LiteX with the VexRiscV CPU
- An On-chip Debugger
- A JTAG programmer
- A FPGA board

### My setup

- **OS** : Ubuntu 20.04
- **LiteX** : [Tag 2022.08](https://github.com/enjoy-digital/litex/tree/2022.08)
- **Toolchain** : riscv64-unknown-elf-gcc
- **On-chip Debugger** : [OpenOCD](https://github.com/SpinalHDL/openocd_riscv) for VexRiscV
- **Board** : Digilent Basys3

### Installations

**riscv64-unknown-elf-gcc toolchain**

```
sudo apt install gcc-riscv64-unknown-elf
```

**OpenOCD for VexRiscV**

If you have any problems installing `OpenOCD`, check the Github [here](https://github.com/SpinalHDL/openocd_riscv).

```
git clone https://github.com/SpinalHDL/openocd_riscv.git
cd openocd_riscv
sudo apt-get install libtool automake libusb-1.0.0-dev texinfo libusb-dev libyaml-dev pkg-config
./bootstrap
./configure --enable-ftdi --enable-dummy
make
sudo make install
```
**SBT**

`sbt` is an open-source build tool for `Scala` and `Java` projects, similar to `Apache's Maven` and `Gradle`, we need it to build the `VexRiscV CPU`.

We will install `Scala` using the `Scala Installer` named [Coursier](https://docs.scala-lang.org/getting-started/index.html).

```
curl -fL https://github.com/coursier/coursier/releases/latest/download/cs-x86_64-pc-linux.gz | gzip -d > cs && chmod +x cs && ./cs setup
```

You should now be able to use the command : `sbt`.

You should now have everything you need to follow the tutorial.

## Build the new VexRiscV CPU

Now we will modify the CPU design parameters to integrate the `JTAG` interface. 

To do this, go to the `pythondata-cpu-vexriscv/pythondata_cpu_vexriscv/verilog` folder and download the `VexRiscV` source code using the following command :

```
cd pythondata-cpu-vexriscv/pythondata_cpu_vexriscv/verilog
git submodule update --init
```

Then you have to modify the build file of the CPU, located at `src/main/scala/vexriscv/GenCoreDefault.scala`.

We need to make some changes :
 1. Add the `imports` needed to add the `JTAG` interface.
 2. Add a new build parameter to specify that you want the interface.
 3. Modify the `DebugPlugin` to connect it to the JTAG interface.

So, first we add the `imports`, we import the specific module `Jtag` from the `spinal.lib.com.jtag` package in the `SpinalHDL` library.

``` scala
package vexriscv

import spinal.core._
import spinal.core.internals.{ExpressionContainer, PhaseAllocateNames, PhaseContext}
import spinal.lib._
import spinal.lib.com.jtag.Jtag // <= Add this line
import spinal.lib.sim.Phase
import vexriscv.ip.{DataCacheConfig, InstructionCacheConfig}
import vexriscv.plugin.CsrAccess.WRITE_ONLY
import vexriscv.plugin._

import scala.collection.mutable.ArrayBuffer

object SpinalConfig extends spinal.core.SpinalConfig(
  defaultConfigForClockDomains = ClockDomainConfig(
    resetKind = spinal.core.SYNC
  )

...
```

Then we add a new build parameter in the script.

``` scala
case class ArgConfig(
  debug : Boolean = false,
  jtag : Boolean = false, // <= Add this line
  hardwareBreakpointCount : Int = 0,
  iCacheSize : Int = 4096,
  dCacheSize : Int = 4096,
  
...

  def main(args: Array[String]) {

    // Allow arguments to be passed ex:
    // sbt compile "run-main vexriscv.GenCoreDefault -d --iCacheSize=1024"
    val parser = new scopt.OptionParser[ArgConfig]("VexRiscvGen") {
      //  ex :-d    or   --debug
      opt[Unit]('d', "debug") action { (_, c) => c.copy(debug = true)   } text("Enable debug")
      opt[Unit]('j', "jtag") action { (_, c) => c.copy(jtag = true)   } text("Add a JTAG interface")
      // ^ Add the line above
...
```

Finally we modify the configuration of the `DebugPlugin` to connect to the `JTAG` interface.

``` scala
...

case plugin: DBusCachedPlugin => {
plugin.dBus.setAsDirectionLess()
master(plugin.dBus.toWishbone()).setName("dBusWishbone")
}
case plugin: DebugPlugin => plugin.debugClockDomain {
    // Add the following lines
    if(argConfig.jtag) {
        plugin.io.bus.setAsDirectionLess()
        val jtag = slave(new Jtag()).setName("jtag")
        jtag <> plugin.io.bus.fromJtag()
    }
}
case _ =>

...
```

You can find the whole source code available on my Github Gist.

{{< github_gist repo="a977819c6b77cc1d5c784361025c8313" >}}

## Modifying the Makefile

Now that we have just modified the `VexRiscV CPU` to include the `JTAG` interface, we just need to add the generation of a new `Verilog` file to the `Makefile`.

``` makefile
SRC := ${shell find . -type f -name \*.scala}

all: VexRiscv.v VexRiscv_Debug.v VexRiscv_Lite.v VexRiscv_LiteDebug.v VexRiscv_LiteDebugHwBP.v VexRiscv_IMAC.v VexRiscv_IMACDebug.v VexRiscv_Min.v VexRiscv_MinDebug.v VexRiscv_MinDebugHwBP.v VexRiscv_Full.v VexRiscv_FullDebug.v VexRiscv_FullCfu.v VexRiscv_FullCfuDebug.v VexRiscv_Linux.v VexRiscv_LinuxDebug.v VexRiscv_LinuxNoDspFmax.v VexRiscv_Secure.v VexRiscv_SecureDebug.v VexRiscv_JTAG.v

# ^ Add "VexRiscv_JTAG.v" to the line above to create the variant of the CPU.

ifeq (,$(wildcard ext/VexRiscv/.github))
$(error Must init/update submodule to get VexRiscv source. Run "git submodule update --init")
endif

...
```

Then we give the rules for generating the new CPU variant at the end of the `Makefile`.

We just use the `-j` parameter to add the `JTAG` interface, you can also modify the `hardwareBreakpointCount` parameter to add new hardware breakpoints. Finally, the new `Verilog` file is called `VexRiscv_JTAG`.

``` makefile
...

VexRiscv_SecureDebug.v: $(SRC)
	sbt compile "runMain vexriscv.GenCoreDefault --pmpRegions 16 --pmpGranularity 256 --csrPluginConfig secure -d --hardwareBreakpointCount 2 --outputFile VexRiscv_SecureDebug"

VexRiscv_JTAG.v: $(SRC)
    sbt compile "runMain vexriscv.GenCoreDefault --csrPluginConfig all -j -d --hardwareBreakpointCount 2 --outputFile VexRiscv_JTAG"

# ^ Add the lines above to create the variant of the CPU.
```

You can find the whole source code available on my Github Gist.

{{< github_gist repo="a977819c6b77cc1d5c784361025c8313" >}}

## Add the new CPU variant into LiteX

Since we have now generated the `Verilog` file of the new variant, we need to integrate it into the `LiteX SoC Generator`, to do this we need to go to the file `litex/litex/soc/cores/cpu/vexriscv/core.py`. This file describes how to integrate the `VexRiscV CPU` variants into the `SoC`, we will modify the CPU connections to connect the `JTAG` interface.

{{< github_gist repo="07d663b7c96ce71492e6fbd71ba13574" >}}

## Resources

{{< github repo="SpinalHDL/openocd_riscv" >}}
<br>
{{< github repo="enjoy-digital/litex" >}}