Common Steps to debug application boot
----------------------------------------

Bootloading an application is a system level issue which is impacted by
hardware as well as software setup. There are some common steps that
users are expected to check while debugging boot related issues. This
section discusses common issues to checkout when debugging application
boot.

|

Hardware Debug Steps
^^^^^^^^^^^^^^^^^^^^^^

.. raw:: html

   <div class="toccolours" style="width:1600px">

**Input clocks and power sequencing**

-  Ensure that correct power sequencing is occurring on your board.
   Power sequencing specifications can be found in the data manual.
-  Check the system clock and ensure this is outputting the expected
   frequency and the signal is swinging rail to rail (should be a 1.8V
   signal). If you have an external oscillator, it should be outputting
   a square wave at the desired frequency. If you are using a crystal ,
   you should have a sine wave at the desired frequency at XTALIN.
   XTALOUT should be similar (may be distorted a little).
-  Check the power on reset signal PORZ. This signal should stay low
   throughout the power sequencing and go high when the power AND high
   frequency clock are stable.
-  If the device has an OBSCLK or CLKOUT then ensure that this pin
   provides the correct output that matches your software
   configurations.

**Boot pins:**

Best way to confirm what the device ROM bootloader reads from the pins
is to read the register where the Boot pins are latched into the device:

-  For AM335x/AM437x: Check the Control_status register
-  For AM57xs: Check CTRL_CORE_BOOTSTRAP register
-  For C66x/K2X devices: Check the DEVSTAT register
-  For OMAPL13x/C674x devices: Check BOOTCFG register

**Boot media connectivity**

Users are required to test their boot interface by running boot loader
independent diagnostic tests that confirm that the SOC can read and
write from the boot interface. Processor SDK RTOS provides Diagnostic
utilities for K2G/AMXX devices and provides POST utility for keystone
devices with which the TI EVM was tested.

TI Technical documentation like Technical reference manual (AMXX
devices), Bootloader USer guide (for C66x/K2X devices) and Bootloader
Application notes for OMAPLxx/C674x list default behavior or the ROM
bootloader and limitations and constraints for each boot mode.
Application developers need to account for this while designing their
system to ensure smooth bring up process.

**Signalling on boot media and reset pins**

If software and other hardware debug steps have not helped identify the
issue, you may need to hook up a scope or protocol analyzer to check the
clocks and the data transfers from the boot media to check if there is
any issues that can be identified. Another commonly known issue that
impacts boot is noise on the power lines which may cause the SOC to
reset or cause a hang on the boot core. Provide these scope shots to TI
on E2E forums so TI engineers can confirm that there are no issues.

.. raw:: html

   </div>

Software Debug Steps
^^^^^^^^^^^^^^^^^^^^^

.. raw:: html

   <div class="toccolours" style="width:1600px">

.. rubric:: SBL and app Entry points
   :name: sbl-and-app-entry-points

If you don`t see your bootloader executing post boot, a good sanity
check is for you to look at the map file for the bootloader and ensure
that the entry point matches with the location in the TI or GPHeader in
the boot image. this can be done by looking at the entry point in the
MLO/_ti.bin/GPheader and ensure that it matches with the location of
symbol Entry in the .out/map file for the bootloader.

.. rubric:: SBL and App memory map
   :name: sbl-and-app-memory-map

The SBL execute from device onchip memory so if your application is also
using Onchip memory in addition to DDR memory, developers need to ensure
that the code sections in the application do not overlap with memory
sections that are used by the SBL. This can cause the SBL to hang during
the application boot process and go into an abort state.

To check what memory region is used by the SBL please locate the map
file created from building the bootloader.

.. rubric:: Flash Programming verification
   :name: flash-programming-verification

It is essential to understand that the flash programmer that you use has
not just the ability to write the image to flash but should also be able
to verify that the image written to flash matches the image that you
have created on your host machine. you can use TI flash programming
utilities as reference and ensure that the production flashing tools use
a method to erase, write and then verify the boot image that is written
to the flash.

|

.. rubric:: DDR timings and configuration /SBL
   :name: ddr-timings-and-configuration-sbl

The Processor SDK bootloader ships with DDR clock and timing settings
that apply to the external memory devices that have been used in TI
evaluation platforms. When running the code on custom platforms, users
may be required to change these setting to match the timings required by
custom design. We highly recommend that users create a GEL script
similar to one provided by TI and test the memory interface with the new
settings before using them in the secondary bootloader to setup external
memory. `**Processor SDK Diagnostics** <index_board.html#diagnostics>`__
provides a mem_test that can be used to test read and writes to the
entire DDR address space for confirming the SOC EMIF settings.

.. rubric:: Emulator based debugging of boot
   :name: emulator-based-debugging-of-boot

The processor SDK RTOS bootloader is like any other application that can
be loaded over the emulator and debugged. Steps to connect an emulator
to the EVM have been described in the :ref:`Hardware Setup Guides <RTOS-SDK-Supported-Platforms>`
that is linked to the Processor SDK RTOS documentation.

Loading the bootloader over emulator can be very useful step in
debugging the system boot. Before generating the final binary for
bootloader, the build generates the .out file for the bootloader which
is then formatted in a boot format that the ROM bootloader(RBL) can
interpret. This .out can be loaded over the emulator similar to any
other application. If you load the debug version of the bootloader, you
can single step through the code that initializes the SOC and also part
of the SBL code that loads the app from the boot media.

.. note::
   SBL runs from onchip memory so users can load the SBL.out even without
   using the GEL file but using the GEL sometimes helps with putting the
   core in a clean state so usage of the GEL is optional


**Alternate Approach:**

**Add a spinlock in SBL main**

In some occasions the RBL may load the SBL but there may be some issue
that you need to debug in the SBL. In this case you can add a spinlock
loop at the start of main in SBL and allow the first stage boot to
complete normally and then connect to the core when you connect the core
will be held in the spinlock condition waiting for you to connect.

Example:

::

    volatile int exitSpinLock = 0;
    void Debug_spinLock(void)
    {
        while (exitSpinLock == 0) { }
    }

You will then have to "release" the core by toggling the variable which
the while loop is blocking on. You can either do this manually by
halting CCS, examining the variable in the watch window and then
changing the value before continuing execution on the core, or you can
use a GEL script.

**Use an IO input to stall the core until user input is received:**

::

    printf("Hit enter to continue:");
    getchar();

.. rubric:: System state at boot failure
   :name: system-state-at-boot-failure

A useful data point for TI to debug boot related issues is to isolate
commonly known initialization and to understand how far the bootloader
has executed correctly before it runs into any issues. Users are
required to capture ARM/DSP clocks, Program counter value, Entry points
detected, pinmux configuration and confirm DDR initialization and target
core states. This helps minimize the number of variables in the system
boot and helps us zero in on the most likely cause for the boot failure.

