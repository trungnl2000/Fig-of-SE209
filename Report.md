# Extra Project

The purpose of this section is to use Quartus® Prime Standard Edition 23.1 and create a basic system with the NiosV/m as the control CPU.

## Target System

![enter image description here](https://raw.githubusercontent.com/trungnl2000/Fig-of-SE209/918b5b9988e86c139cc039e976c068d27d6b396a/niosv_system_top_level.png)

It is composed of the following elements:

-   A Nios V/m core in base profile.
-   An embedded memory using the onboard memory blocks of the FPGA.
-   A serial communication interface:
    -   It will be seen as a UART and can be used for Standard I/O
    -   physically, it will use the USB link used to program the FPGA
    -   and during code execution, it will serve as the interface of the Debug.
-   Generic I/O modules (GPIOs):
    -   as an example of IP integration,
    -   They will be connected to the switches and LEDs on the board
    -   and will test the program run by the processor.

We will go through the following steps:

1.  Build the system in Qsys/Platform Designer
2.  Integrate this system into the FPGA stream (Quartus)
3.  Write test software to test the operation of the Devices

## System construction

To simplify integration with the FPGA synthesis stream, we start from a pre-built project, which can be found [here](https://perso.telecom-paristech.fr/graba/se209/lab1/de1-soc.tgz). This draft contains the following information:

-   The FPGA target (here a Cyclone V - 5CSEMA5F31C6) (*DE1_SoC.qpf*),
-   An RTL module (in SystemVerilog) with associated inputs/outputs map elements (*DE1_SoC.sv*),
-   The correspondence between the inputs/outputs of the RTL module and the FPGA pins (*DE1_SoC.qsf*),
-   A file describing the clocks and constraints associated with the  _SDC_  (Synopsys Design Constraints) format (*DE1_SoC.sdc*).


> _Platform Designer_  can be used directly independently of Quartus. In this case, the target FPGA will have to be specified and add mapping information, and then go back to Quartus to synthesis.

Once the archive is unzipped, you can open the project in the  _Quartus® Prime Standard Edition 23.1_  software:
```File -> Open Project... -> Navigate to DE1_SoC.qpf -> Open```
![enter image description here](https://github.com/trungnl2000/Fig-of-SE209/blob/main/Open%20Project%20in%20quartus.png?raw=true)
This project contains a unique RTL (`DE1-SoC.sv`) file with the Inputs/Outputs definitions. Some outputs have been assigned to constant values (for example, the LEDs are turned off). This module (`DE1_SoC`) is the top level block of our design and will contain later the SoC system that we will design in _Platform Designer_.

To open _Platform Designer_, from the _Tools_ menu, select _Platform Designer_. This will create an empty integration project and should look as follows.

![enter image description here](https://github.com/trungnl2000/Fig-of-SE209/blob/main/empty%20project%20in%20platform%20designer.png?raw=true)

This is the starting point to construct our system.

The newly created system contains a clock and Reset submodule. Note that in the  _Export_  column we have two signals (`clk`  and`reset`) that are exported which means that we will be able to connect them to the clock and reset source in the RTL top design.

# Adding IP blocks

The left lateral pane of the Platform Designer window should contain an  `IP Catalog`  tab. This represents the Library of IPs that are available and hat we can use in our design. The different blocks are organized in browsable categories and a search field helps to find specific blocks. 

## Adding the Processor Core
The first element that we will add is the `Nios V/m` processor. You can find it in the `Processors and Peripheral/Embedded Processors` section. Or by simply typing Nios V in the search bar, and double click on *Nios V/m Microcontroller Intel FPGA IP*. 
![enter image description here](https://github.com/trungnl2000/Fig-of-SE209/blob/main/niosv%20search.png?raw=true)
Then, a window appear, make sure check the option *Enable Reset from Debug Module*.
![enter image description here](https://github.com/trungnl2000/Fig-of-SE209/blob/main/Add%20NiosV.png?raw=true)

## Adding the On-Chip Memory block
The second element that we will integrate in the design is an On-Chip Memory. It can be found in the  `Basic Function` or by typing "ram" in the search bar.

This IP will use the FPGA embedded memory block and as they can be initialized with the FPGA bitstream, it can act as both Read Only Memory (ROM) for the program and constants and Random Access Memory (RAM) for data and variable.

Change the Total Memory Size configuration to 128 KB as shown in the following figure. This will allow using a simple C library with the  `printf`  function.

![enter image description here](https://github.com/trungnl2000/Fig-of-SE209/blob/main/add%20ram.png?raw=true)
## Adding the GPIO controllers

Then we add two General Purpose Input/Output controllers (search for  `Parallel I/O (PIO)`). The PIO IP can be configured in width and direction.

The first one will be used to attach to the 10 LEDs (`ledr`  output in the RTL top).

Modify the configuration of the  _PIO_  IP to have 10 outputs pins as depicted here.

![](https://perso.telecom-paristech.fr/graba/ics901/platform-designer-tutorial/qsys_pio.png)

The second PIO will be connected to the 10 switches (`sw`  signal in the RTL top). Thus, add a second PIO bloc with 10 pins configured as inputs.

## Adding the JTAG UART

This last element can be found in  `Serial`  subsection of  `Interface Protocols`  section. Keep the default configuration this time.

After that, the schematic of our system should look like in the following figure:
![enter image description here](https://github.com/trungnl2000/Fig-of-SE209/blob/main/raw%20schematic.png?raw=true)

# Connecting the different elements

The connection’s matrix is visible in the  `Connections`  column of the main window. The white dots represent possible connections. When clicked, they turn black to signify that the connection is effective.

The tool does not allow connections between incompatible interfaces (you can not connect an output clock to a reset input for example).

Here are the connections that we want to have in our system:

-   The clock input of all the blocks should be connected the clock output of the clock source
-   The reset input of all the blocks should be connected the reset output of the clock source
-   The slave interface of the on-chip ram should be connected to both the data and instruction manager (master) of the processor. This will allow the processor to fetch instructions (program) and data (variables) from the same (and unique here) memory
-   The slave interface of all other blocks should be connected to the sole data manager interface from the processor because the processor does not fetch instructions from those peripherals
-   Connect the  `irq`  (interrupt request) output from the JTAG UART to the processor interrupt receiver.

All connections done, the connection’s matrix should look as follows.
![enter image description here](https://github.com/trungnl2000/Fig-of-SE209/blob/main/connected%20schematic.png?raw=true)

**Remarks**

-   To change the name of one of the instantiated IPs, you can select it and press  `CTRL-R`
-   The Nios V/m core can have up to 16 IRQs (0 to 15). You can change the IRQ index by changing the corresponding index in the  `IRQ`  column (here we have chosen to connect the JTAG IRQ irq to the irq number 0 of the processor).

# Assigning peripherals base addresses

The column  `BASE`  show the base address of each peripheral. For the moment, all peripherals have a base address equal to  `0`.

We can assign the base addresses manually, but we must take care of avoiding overlapping ranges.

The tool provides a way to assign those addresses automatically through the item  `Assign Base Adresses`  of the  `System`  menu.

![](https://perso.telecom-paristech.fr/graba/ics901/platform-designer-tutorial/qsys_assign_base.png)

We want to keep the base address of the On-chip Memory at the value  `0`  even with the automatic assignation of base addresses. We can lock this parameter by setting the lock symbol in the corresponding line of the Base column. Then we can reassign base addresses automatically.

The final address mapping should look like the following (some differences can exist depending on the order of appearance of the different IPs).

![enter image description here](https://raw.githubusercontent.com/trungnl2000/Fig-of-SE209/main/assigned%20base%20adress%20schematic.png)

## Defining the reset and exception vectors of the processor

Now that the different addresses of the peripherals are set, we can go back to the processor configuration and set the reset and exception vectors. Those addresses correspond, respectively, to the address of the first instruction executed after a reset (or at the boot), and the address of the exception/interrupt handler.

By double-clicking on the processor in the main window, the configuration pane should appear. In the  `Verctors`  section you can specify the reset agent.

Here, we choose the On-Chip Memory (SRAM.s1) instead of giving an absolute value, as it allows to change the mapping and size of the memory without loosing track:

![enter image description here](https://github.com/trungnl2000/Fig-of-SE209/blob/main/set%20vector%20offset%20cpu.png?raw=true)

Normally, at this step, you should not have any error or warning message.

# Exporting the inputs and outputs

The inputs and outputs from the two GPIO controllers needs to be connected to the corresponding pins of the FPGA. To achieve this, the signals must be exported from the system to be connected in the top level RTL module.

In the main pane, for each PIO, you can export the external connection, by double-clicking in the export column. There, you can choose a explicit name, for example, here  `leds`  and  `sw`  as it is shown in the following figure.

![enter image description here](https://github.com/trungnl2000/Fig-of-SE209/blob/main/exported%20IO.png?raw=true)

# Saving the project and generating the RTL

The last step, before going back to Quartus and synthesising the design, is to generate RTL files for the current project.

The first thing to do, is saving the project by choosing the  `Save`  item in the  `File`  menu. Here you can choose a explicit name for the system you have build (for example  `niosv_system`).

![enter image description here](https://github.com/trungnl2000/Fig-of-SE209/blob/main/saved%20schematic.png?raw=true)

To generate the RTL, choose the  `Generate HDL`  from the  `Generate`  menu.

![enter image description here](https://github.com/trungnl2000/Fig-of-SE209/blob/main/generate%20schematic.png?raw=true)

A directory with the same name as your system will be created in the Quartus project directory. Take some time to analyse the content of this directory.

# Integration with the Quartus project and test on the board

You can now quit Platform Designer. A message will give you instructions on how to add the design to the Quartus Project:
![enter image description here](https://github.com/trungnl2000/Fig-of-SE209/blob/main/missing%20ip.png?raw=true)
To add the new IP Variation to your project: In Quartus, select `Add/Remove files in project...` from the `Project` menu. Open the file chooser by clicking on the `...` icon and choose the `.qip`  file (only the `.qip` file) corresponding to your system as described in the figure above.

Once done, you must modify the top level Verilog file by adding an instance of the system. A  `_inst.v`  file (`niosv_system/niosv_system_inst.v`) should have also been created with an example of how to instantiate your design.

We will use the  `clock_50`  input as the main clock of our system and the  `key[0]`  as reset signal. We will also connect the exported signals from the PIO to the corresponding switches and leds signals.

Here is en example of the instantiation:

```
    niosv_system u0 (
        .clk_clk       ( clock_50 ) ,
        .reset_reset_n ( key[0]   ) ,
        .sw_export     ( sw       ) ,
        .leds_export   ( ledr     )
    );
```

**Note**

-   As the  `ledr`  output is already assigned to a constant, you must remove (or comment) this assignment.

Once the instance is added, you can compile the design in Quartus by press `Ctrl+L`. This will run the synthesis and generate the  _bitstream_  to program the FPGA.

# Testing on the real hardware


This section provides the design flow to generate and build a Nios V/m processor software project. Steps on how to generate an Application and Board Support Package (BSP) project using the `niosv-app` and `nios-bsp` utilities are provided.

## Generating the Board Support Package
**Step 1:** Open niosv shell: Open a new terminal window by using `niosv-shell` provided by our school:

`/comelec/softs/opt/altera/altera23.1std/niosv/bin/niosv-shell`

**Step 2:** Generate BSP:
`/comelec/softs/opt/altera/altera23.1std/niosv/bin/niosv-bsp  -c  -t=hal  -sopcinfo=niosv_system.sopcinfo  software/bsp/settings.bsp`

Where:
-   `/comelec/softs/opt/altera/altera23.1std/niosv/bin/niosv-bsp` is the command to open the niosv-bsp program.
-   `-c` means create BSP (Board Support Package).
-   `-t=hal` specifies the creation of a Hardware Abstraction Layer (HAL), which provides a consistent interface to the hardware.
-   `-sopcinfo=niosv_system.sopcinfo` specifies the SOPC (System on a Programmable Chip) information file, which contains details about the system configuration. (Change `niosv_system.sopcinfo` to the name of your system, which was generated by Platform Designer in the previous step)
-   `software/bsp/settings.bsp` specifies the BSP settings file, which contains the configuration for the Board Support Package.

**Step 3:** Create necessary folder and file:
```
mkdir  software/app
touch  software/app/gpio-test.c
```
Where `gpio-test.c` is an arbitrary name for our test program.

**Step 4:** Generate Cmake for our application:
`/comelec/softs/opt/altera/altera23.1std/niosv/bin/niosv-app  -a=software/app  -b=software/bsp  -s=software/app/gpio-test.c` 

Where:
-   `/comelec/softs/opt/altera/altera23.1std/niosv/bin/niosv-app` is the command to open the niosv-app program.
-   `-a=software/app` specifies the application directory.
-   `-b=software/bsp` specifies the Board Support Package (BSP) directory.
-   `-s=software/app/gpio-test.c` specifies the source file for the application.

**Step 5:** Open RiscFree IDE: 
In this tutorial, RiscFree IDE will be used. It is a tool created by Ashling Microsystems, designed for embedded software development. RiscFree IDE supports various processor architectures and provides features such as code editing, debugging, and project management.
`/comelec/softs/opt/altera/altera23.1std/riscfree/RiscFree/RiscFree`
Make sure to run this command within niosv-shell
**Step 6:** Choose workspace:
Navigate to the created software folder:
![enter image description here](https://github.com/trungnl2000/Fig-of-SE209/blob/main/riscfree%20workspace.png?raw=true)
**Step 7:**  Create a new project
![enter image description here](https://github.com/trungnl2000/Fig-of-SE209/blob/main/create%20c%20project.png?raw=true)
![enter image description here](https://raw.githubusercontent.com/trungnl2000/Fig-of-SE209/main/create%20riscfree%20project.png)
**Step 8:** Finish the program:
![enter image description here](https://github.com/trungnl2000/Fig-of-SE209/blob/main/code%20c%20in%20riscfree.png?raw=true)
**Remark:** `0x00030050` and `0x00030040` are the addresses assigned to the LEDS and SW modules, respectively, by Platform Designer in the previous step (Change them if you have different addresses).

**Step 9:** Left click on project (here is `app`), choose `Build Project`
**Step 10:** Go back to Quartus, select `Programmer` in the `Tools` menu, a new window appears, then follow the instructions in the following figures to re-program your kit. Make sure to connect it to your computer before doing anything:
![enter image description here](https://github.com/trungnl2000/Fig-of-SE209/blob/main/reconfig%20fpga%20step2.png?raw=true)
![enter image description here](https://github.com/trungnl2000/Fig-of-SE209/blob/main/reconfig%20fpga%20step3.png?raw=true)

> 2: Double click on DE-SoC

![enter image description here](https://github.com/trungnl2000/Fig-of-SE209/blob/main/reconfig%20fpga%20step4.png?raw=true)
![enter image description here](https://github.com/trungnl2000/Fig-of-SE209/blob/main/reconfig%20fpga%20step6.png?raw=true)
> 7: Double click and navigate to DE1_SoC.sof, which was generated by the compilation process in Quartus.

**Step 11:** Go back to RiscFree IDE, `left click on the project (here is app) -> Run As -> 3 Ashling RISC-V Hardware Debugging`, then follow:
![enter image description here](https://github.com/trungnl2000/Fig-of-SE209/blob/main/choose%20elf%20file%20in%20riscfree.png?raw=true)
![enter image description here](https://github.com/trungnl2000/Fig-of-SE209/blob/main/choose%20debugger.png?raw=true)
**Step 12:** At this step, you can turn the switches on and off to check the behavior of the LEDs. If you want to see the program's output on the terminal, open a new terminal tab and use: `/comelec/softs/opt/altera/altera23.1std/quartus/bin/juart-terminal`

You should see the following result:
![enter image description here](https://github.com/trungnl2000/Fig-of-SE209/blob/main/result.png?raw=true)
