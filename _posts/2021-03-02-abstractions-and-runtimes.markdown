---
layout: post
title: "Abstractions and Runtimes"
date: 2021-03-02 18:37:00 -0800
categories: rp2040 pico ada
---
A month later, much has happened! I've implemented drivers for nearly all of the RP2040's peripherals, [Fabien Chouteau](https://github.com/Fabien-Chouteau) contributed an I2C driver, and [Daniel King](https://github.com/damaki) has ported bb-runtimes, including multiprocessing support. Let's get that blink example updated!

There are lots of pieces that can be abstracted away and made more flexible. For example, `Ticks` shouldn't be a public variable that can be modified from anywhere. We should probably also define a new `Time` type to differentiate it from other Integers. If you follow these changes to their logical conclusion, you get something that looks like [Ada.Real_Time](https://www.adaic.org/resources/add_content/standards/05aarm/html/AA-D-8.html) which already exists in the Ravenscar runtimes.

The Ada language provides some fairly high level constructs related to tasking, memory management, and timing that aren't easy or practical to implement on every platform and the use of some of those features may violate a project's certification requirements. For this reason, there several runtime profiles with varying levels of functionality. So far, we've been using a Zero Footprint (ZFP) runtime, which provides only the bare minimum to allocate some stack space, pass arguments, and call a procedure. No batteries included. The next step up would be the [Ravenscar](http://ada-auth.org/standards/12rm/html/RM-D-13.html) profile, which includes a broader set of builtin functionality, including tasking and synchronization constructs you'd expect to find in an RTOS. An open source implementation of Ravenscar is available in the [bb-runtimes](https://github.com/AdaCore/bb-runtimes) repository, though porting it to a new chip is not a small task. There are other runtime implementations that wrap existing RTOS libraries like [FreeRTOS](https://github.com/simonjwright/cortex-gnat-rts) and [RTEMS](https://devel.rtems.org/wiki/TBR/UserManual/RTEMSAda). GNAT GCC also includes runtimes for Linux, FreeBSD, Solaris, HP-UX, VxWorks, LynxOS, QNX, and Windows that implement the full set of libraries in the Ada language specification.

Let's try porting our blink example to a Ravenscar RTS (Run-Time System) from bb-runtimes. As the RP2040 is still a new platform, it's runtimes haven't been merged yet and aren't distributed in the GNAT Community 2020 bundle so we'll need to build it from source.

    git clone -b rpi-pico https://github.com/damaki/bb-runtimes
    cd bb-runtimes
    ./build_rts.py --build rpi-pico-mp
    cd ../02-ravenscar-blink
    gprbuild -P rpsimple.gpr

You'll notice that the boot2, crt0, and linker script aren't needed anymore as they're included with the runtime. I've replaced all of the SysTick stuff with a single `delay 1.0;` statement. The RTS has configured the PLLs and the TIMER peripheral, so we now have microsecond resolution tickless operation. We can still do better. Our single-cycle loop still takes time to execute, and waking from sleep takes a few cycles too, so the delay between blinks is still not precisely one second.

In [03-realtime-blink](03-realtime-blink/src/main.adb) I've imported `Ada.Real_Time` and declared a `Next_Blink` variable with type `Time`. Time is a private type as the storage representation of time is a non-portable implementation detail. Initializing `Next_Blink` with a call to the `Clock` function means that time doesn't even have to start at 0!

    Next_Blink : Time := Clock;

Now that we know when the next blink should happen, we can use a `delay until` statement for precision waiting.

    loop
       SIO_Periph.GPIO_OUT_XOR.GPIO_OUT_XOR := Pin_Mask;
       delay until Next_Blink;
       Next_Blink := Next_Blink + Seconds (1);
    end loop;
