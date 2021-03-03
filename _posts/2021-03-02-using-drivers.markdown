---
layout: post
title: "Using drivers"
date: 2021-03-02 18:38:00 -0800
categories: rp2040 pico ada
---
Twiddling bits in registers doesn't make the most intuitive, readable, or portable code. This is why we write drivers. I've created three [Alire](https://alire.ada.dev/) packages, [rp2040_hal](https://github.com/JeremyGrosser/rp2040_hal), [pico_bsp](https://github.com/JeremyGrosser/pico_bsp), and [pico_examples](https://github.com/JeremyGrosser/pico_examples). rp2040_hal contains all of the drivers for the chip's internal peripherals, pico_bsp contains some details about the Pico board and drivers for the Pimoroni Pico addons, and pico_examples contains, you guessed it, example code. At the moment, pico_bsp cannot be used with the Ravenscar runtime without modification, so we'll ignore that and focus on rp2040_hal for right now. The examples repository contains lots of code that uses the pico_bsp if you'd like to see how that works.

Now that we have drivers, I've removed all of the interfaces generated from SVD from our project. These still live in the rp2400_hal package but as long as the drivers implement all of the right interfaces, we shouldn't need them. If you find something you can't do with the HAL drivers that seems important, please send a pull request to the rp2040_hal repo.

	with Ada.Real_Time; use Ada.Real_Time;
	with RP.GPIO; use RP.GPIO;
	with RP.Clock;

	procedure Main is
		LED        : GPIO_Point := (Pin => 25);
		Next_Blink : Time := Clock;
	begin
		RP.Clock.Initialize (XOSC_Frequency => 12_000_000);
		LED.Configure (Output);

		loop
		   LED.Toggle;
		   delay until Next_Blink;
		   Next_Blink := Next_Blink + Seconds (1);
		end loop;
	end Main;

You can see that our Main procedure is much more compact and readable now. The clock and PLLs get configured with external oscillator frequency, the PADS_BANK and IO_BANK stuff is wrapped up in a lovely Configure interface, and our GPIO itself is abstracted into an object with a very convenient Toggle method.

Building the code is done using Alire now, rather than calling gprbuild directly. Alire is analogous to Rust's Cargo or Python's pip. Alire keeps track of all of the dependencies and allows us to pull in new ones with the `alr with` command. At the time of writing, the rp2040_hal package isn't available in the Alire index yet, you can clone it manually and pin the dependency.

	git clone https://github.com/JeremyGrosser/rp2040_hal
	cd 04-hal-blink
	alr pin --use=../rp2040_hal rp2040_hal
	alr build
