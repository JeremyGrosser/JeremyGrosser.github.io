---
layout: post
title: "Making Noise with Ada"
date: 2021-04-02 14:00:00 -0800
categories: ada pico
---
I'm building an audio synthesizer with Ada on the [Raspberry Pi Pico](https://www.raspberrypi.org/products/raspberry-pi-pico/)'s [RP2040](https://www.raspberrypi.org/documentation/rp2040/getting-started/) chip. See my [earlier](https://synack.me/ada/pico/2021/03/03/from-zero-to-blinky-ada.html) [posts](https://synack.me/ada/pico/2021/03/03/abstractions-and-runtimes.html) about board bring up. This chip has 256 KB of memory, no floating point unit, and no DAC or I2S peripheral. But, it does have some very flexible I/O state machines and a well optimized soft float library in ROM.

There's a PIO program in pico-extras called [audio_i2s.pio](https://github.com/raspberrypi/pico-extras/blob/master/src/rp2_common/pico_audio_i2s/audio_i2s.pio) that handles the timing sensitive clocking and data bits of I2S. All I need to do is push 16-bit two's complement signed integers into the PIO FIFO and the state machine handles the rest!

The PIO program includes a block of C code that calls pico-sdk to configure the state machine's pin muxing and jump to the program's entry point. I stumbled a bit trying to rewrite this in Ada, but things went much smoother after I refactored my RP.PIO driver to more closely match pico-sdk's interfaces. This code populates a Configuration record with a bunch of settings, then applies them to the PIO both by setting registers directly and executing a few instructions on the state machine.

[pico-audio_i2s.adb](https://github.com/JeremyGrosser/pico_bsp/blob/master/src/pico-audio_i2s.adb)
{% highlight ada %}
   procedure Program_Init
      (This   : in out I2S_Device;
       Offset : PIO_Address)
   is
   begin
      This.Config := Default_SM_Config;
      Set_Out_Pins (This.Config, This.Data.Pin, 1);
      Set_Sideset_Pins (This.Config, This.BCLK.Pin);
      Set_Sideset (This.Config, 2, False, False);
      Set_Out_Shift (This.Config, False, True, 32);
      Set_Wrap (This.Config,
          Wrap        => Offset + Pico.Audio_I2S_PIO.Audio_I2s_Wrap,
          Wrap_Target => Offset + Pico.Audio_I2S_PIO.Audio_I2s_Wrap_Target);

      Set_Config (This.PIO.all, This.SM, This.Config);
      SM_Initialize (This.PIO.all, This.SM, Offset, This.Config);

      Set_Pin_Direction (This.PIO.all, This.SM, This.Data.Pin, Output);
      Set_Pin_Direction (This.PIO.all, This.SM, This.BCLK.Pin, Output);
      Set_Pin_Direction (This.PIO.all, This.SM, This.LRCLK.Pin, Output);

      Execute (This.PIO.all, This.SM, PIO_Instruction (Offset + Pico.Audio_I2S_PIO.Offset_entry_point));
   end Program_Init;
{% endhighlight %}

The rest of the clock and GPIO configuration for the PIO is pretty straightforward and I was able to get things up and running pretty easily... Except for my Pico's broken internal pull down resistor, which took me a while to debug and ended with a silly looking 0603 resistor pulling the I2S data pin to GND.

![Pico pull down resistor fix](/img/pulldown-fix.jpg)

The RP.PIO driver only supports blocking writes to the FIFO, which wastes a lot of cycles that could be used to generate more interesting audio. I had never implemented a DMA driver before and was expecting a challenge, but it was suprisingly simple! Load some configuration registers, set a source and destination address and buffer size, and pull the trigger. The RP2040's DMA channels have a CTRL register that is aliased four times at different memory addresses and the triggering behavior is slightly different depending on which one you write to. This was a little difficult to understand from the docs, but I eventually figured it out.

[rp-dma.adb](https://github.com/JeremyGrosser/rp2040_hal/blob/master/src/drivers/rp-dma.adb#L23)
{% highlight ada %}
   procedure Configure
      (Channel : DMA_Channel_Id;
       Config  : DMA_Configuration)
   is
   begin
      DMA_Periph.CH (Channel).AL1_CTRL :=
         (EN            => True,
          HIGH_PRIORITY => Config.High_Priority,
          DATA_SIZE     => Config.Data_Size,
          INCR_READ     => Config.Increment_Read,
          INCR_WRITE    => Config.Increment_Write,
          RING_SIZE     => Config.Ring_Size,
          RING_SEL      => Config.Ring_Wrap,
          CHAIN_TO      => Config.Chain_To,
          TREQ_SEL      => Config.Trigger,
          IRQ_QUIET     => Config.Quiet,
          BSWAP         => Config.Byte_Swap,
          SNIFF_EN      => Config.Sniff,
          others        => <>);
   end Configure;

   procedure Start
      (Channel  : DMA_Channel_Id;
       From, To : System.Address;
       Count    : HAL.UInt32)
   is
   begin
      DMA_Periph.CH (Channel).READ_ADDR := From;
      DMA_Periph.CH (Channel).WRITE_ADDR := To;
      DMA_Periph.CH (Channel).AL1_TRANS_COUNT_TRIG := Count;
   end Start;
{% endhighlight %}

I then rewrote the `Pico.Audio_I2S.Transmit` method to use DMA.

[pico-audio_i2s.adb](https://github.com/JeremyGrosser/pico_bsp/blob/master/src/pico-audio_i2s.adb)
{% highlight ada %}
   overriding
   procedure Transmit
      (This : in out I2S_Device;
       Data : HAL.Audio.Audio_Buffer)
   is
      Count : HAL.UInt32 := Data'Length;
   begin
      --  Wait for previous DMA transfer to finish before modifying the buffer.
      while Busy (This.DMA_Channel) loop
         null;
      end loop;

      This.Buffer (1 .. Data'Length) := Data;

      if This.Channels > 1 then
         Count := Data'Length / 2;
      end if;

      RP.DMA.Start
         (Channel => This.DMA_Channel,
          From    => This.Buffer'Address,
          To      => TX_FIFO_Address (This.PIO.all, This.SM),
          Count   => Count);
   end Transmit;
{% endhighlight %}

`HAL.Audio.Audio_Buffer` is an array of 16-bit integers. If stereo audio is used, the samples are interleaved. The PIO FIFO buffer is 32 bits wide and the PIO program expects two 16-bit samples in each FIFO write, one for each channel. For mono audio, the top 16 bits are zeroed. The DMA channel is configured for 16 or 32 bit writes depending on `This.Channels`. As long as Data is 32 bit aligned, this works out perfectly.

After I got DMA working, I spent a few days on audio synthesis. I'd never written any software to generate audio before, so I had a bit of domain knowledge to catch up on but now that I understand it, it's pretty straightforward. The only real complication is the need to call the RP2040's ROM floating point routines if performance or code size is a concern. Daniel King already did [most of the hard work](https://github.com/damaki/bb-runtimes/blob/rpi-pico/arm/rpi/rp2040/s-bootro.adb#L42) in making those library symbols available in Ada and my [RP.ROM.Floating_Point](https://github.com/JeremyGrosser/rp2040_hal/blob/master/src/drivers/rp-rom-floating_point.ads) mostly follows the same pattern.

Neither bb-runtimes nor rp2040_hal override gcc's trigonometry functions (sinf, cosf, tanf, etc) with the ROM's implementations as the ROM's floating point library isn't strictly compatible with what gcc expects. It would likely work for our use case, but I don't want to make that decision for other people. Instead, if you need the speed of the ROM functions, you can call them directly. Note that this is different from pico-sdk's behavior, which uses the ROM functions by default.

Now I have a [working wavetable oscillator](https://github.com/JeremyGrosser/pico_examples/tree/master/pimoroni_audio_pack/src), ADSR envelope filter, and IIR low pass filter on the Pico. I'm starting to think about what kind of user interface I want on this synthesizer and think it would be neat to use the [Pimoroni RGB Keypad](https://shop.pimoroni.com/products/pico-rgb-keypad-base) as a sort of crossbar selector for connecting four oscillators to four filters. Unfortunately, the way the Pico Audio Pack is designed, I can't have it plugged into a Pico at the same time as the keypad, so I'm probably going to need to design a PCB to get it all wired up.

The [PCM5100A](https://www.ti.com/product/PCM5100A) I2S DAC that the audio pack uses looks to be in short supply at the moment, but its older sibling, [PCM1754](https://www.ti.com/product/PCM1754) is cheap and widely available. The most significant difference between the two is that the PCM1754 needs a MCLK signal running at a multiple of BCLK, whereas the PCM5100A generates MCLK internally by doing clock recovery from BCLK with a PLL. I think the easiest way to generate MCLK from the RP2040 would be to run a second PIO state machine at the higher clock speed and start it in sync with the I2S state machine. The MCLK PIO program should be very simple, just toggling MCLK on each cycle.

Hopefully next month I'll have some fun sounds and pretty PCB layouts to show you!
