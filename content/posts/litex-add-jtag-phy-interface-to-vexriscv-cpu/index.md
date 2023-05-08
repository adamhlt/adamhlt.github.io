---
title: "LiteX - Add JTAG PHY interface to VexRiscV CPU"
date: 2023-02-23T17:54:41+02:00
draft: false
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

First, we will add a new variant to use the correct CPU `Verilog` file. To do this we will add 1 new entry to the list of variants, `CPU_VARIANTS` and also to the list of compiler parameters, `GCC_FLAGS`. Since our new variant is based on the `full` variant we will call our variant `full+jtag` to recognise it. Finally we come to use the same compiler parameters as the `full` variant.

```python
...

CPU_VARIANTS = {
    
    ...

    "secure":             "VexRiscv_Secure",
    "secure+debug":       "VexRiscv_SecureDebug",
    "full+jtag":          "VexRiscv_JTAG",
    # The line above specifies which Verilog file to use.
}

# GCC Flags ----------------------------------------------------------------------------------------

GCC_FLAGS = {
    
    ...

    "secure":           "-march=rv32i2p0_ma    -mabi=ilp32",
    "secure+debug":     "-march=rv32i2p0_ma    -mabi=ilp32",
    "full+jtag":        "-march=rv32i2p0_m     -mabi=ilp32", 
    # The line above specifies the GCC compilation parameters.
}

...
```

Finally, we need to add the connection of the `JTAG` signals to the rest of the `SoC`. To do this we will detect the presence of the keyword `jtag` in the variant name and then call a function that will connect the signals correctly.

```python
...

        # Add Timer (Optional).
        if with_timer:
            self.add_timer()

        # Add Debug (Optional).
        if "debug" in variant:
            self.add_debug()

        if "jtag" in variant:
            self.add_jtag()

    def set_reset_address(self, reset_address):
        self.reset_address = reset_address
        self.cpu_params.update(i_externalResetVector=Signal(32, reset=reset_address))
...
```

```python
...

            o_debug_bus_cmd_ready           = self.o_cmd_ready,
            o_debug_bus_rsp_data            = self.o_rsp_data,
            o_debug_resetOut                = self.o_resetOut
        )

    def add_jtag(self):
        debug_reset = Signal()

        self.i_jtag_tdi = Signal()
        self.i_jtag_tms = Signal()
        self.i_jtag_tck = Signal()
        self.o_jtag_tdo = Signal()
        self.o_resetOut = Signal()

        reset_debug_logic = Signal()

        self.sync += debug_reset.eq(reset_debug_logic | ResetSignal())
    
        self.cpu_params.update(
            i_reset = ResetSignal() | self.reset | debug_reset,
            i_debugReset     = ResetSignal(),
            i_jtag_tdi       = self.i_jtag_tdi,
            i_jtag_tms       = self.i_jtag_tms,
            i_jtag_tck       = self.i_jtag_tck,
            o_jtag_tdo       = self.o_jtag_tdo,
            o_debug_resetOut = self.o_resetOut
        )


    def add_cfu(self, cfu_filename):
        # Check CFU presence.
        if not os.path.exists(cfu_filename):

...
```

You can find the whole modified `core.py` file on the Github Gist.

{{< github_gist repo="a977819c6b77cc1d5c784361025c8313" >}}

## Connect the JTAG interface

Everything is now ready, all that is left to do is to create a simple `Migen` component to connect the `JTAG` interface to the `IOs` on the FPGA board, to allow us to use a `JTAG` debugger.

We can now create the new component `jtag_phy.py` in the folder `litex/litex/soc/cores`. This component is just to connect the `JTAG` interface to the pins of the board we will declare in the `platform` file.

Here is the component `jtag_phy.py` :

```python
from migen import *

class JTAG(Module):
    def __init__(self, soc, pads):
            
        # Connect JTAG interface for VexRiscV
        # Check if the VexRiscV CPU has a JTAG interface
        if(soc.cpu_type is "vexriscv" and soc.cpu_variant is not None and "jtag" in soc.cpu_variant):
            
            # Connect the JTAG interface of the CPU to the GPIOs
            self.comb += [
                soc.cpu.cpu_params["i_jtag_tdi"].eq(pads.tdi),
                soc.cpu.cpu_params["i_jtag_tms"].eq(pads.tms),
                soc.cpu.cpu_params["i_jtag_tck"].eq(pads.tck),
                pads.tdo.eq(soc.cpu.cpu_params["o_jtag_tdo"])
            ]

            return
```

You must also modify the `platform` file of your card to add the `JTAG` pins, here is an example for the `Basys3` board :

```python
("jtag", 0,
    Subsignal("tms", Pins("J1")),
    Subsignal("tdi", Pins("L2")),
    Subsignal("tdo", Pins("J2")),
    Subsignal("tck", Pins("G2")),
    IOStandard("LVCMOS33"),
),
```

Finally you can import the component in the `target` file like this: `from litex.soc.cores.jtag_phy import JTAG` and connect the interface by adding a submodule: `self.submodules.jtag = JTAG(self, platform.request("jtag"))`. You can find the whole source code on the Github Gist.

{{< alert cardColor="#d86c46" iconColor="#f1faee" textColor="#f1faee" >}}
Don't forget to add the following line into the target file to avoid error about clock route.
{{< /alert >}}

```python
platform.add_platform_command("set_property CLOCK_DEDICATED_ROUTE FALSE [get_nets jtag_tck_IBUF]")
```

{{< github_gist repo="07d663b7c96ce71492e6fbd71ba13574" >}}

## Build the SoC and debug the CPU

Now you can build the SoC with the `JTAG` interface with the following command: 

```console
./digilent_basys3.py --cpu-variant=full+jtag --build
```

![Build Log](https://user-images.githubusercontent.com/48086737/236834622-98437ce6-5912-41da-b3a4-858480e778c9.png "Build of the LiteX SoC")

After you have built your SoC you can retrieve the `.yaml` file of the CPU configuration, this file is in the same place as the `Verilog` file of the CPU. Then download the script to connect to the `JTAG` interface of the CPU using `OpenOCD`.

{{< github_gist repo="d06306e520f3f2ae46e8c3aa08a40dd6" >}}

<br>

{{< alert >}}
You have to make sure that the script matches your configuration and the JTAG debugger you are using. In my case I use a **Digilent HS2**.
{{< /alert >}}

Finally, you can run the script, if everything goes well you will not get any errors and you can open a connection with `GDB`.

![Basys3](https://user-images.githubusercontent.com/48086737/236834803-c10fa757-526d-47ac-8a72-15c5be73dee6.png "Basys3 and Digilent HS2 setup.")

<img width="1150" alt="Capture d’écran 2023-05-08 à 14 37 53" src="">

![OpenOCD](https://user-images.githubusercontent.com/48086737/236834943-93e955f9-da30-472b-bd62-a308c1dbcf89.png "OpenOCD connection ready for debug.")

## Resources

{{< github repo="SpinalHDL/openocd_riscv" >}}

<br>

{{< github repo="enjoy-digital/litex" >}}
