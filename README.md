# Dreamcast Pop'n'Music Controller<br/>Using Raspbery Pi Pico (RP2040)

## Intro
This is a homebrew controller for playing the Pop'n'Music games on the Sega Dreamcast using a Raspberry Pi Pico.

![Image of controller](https://github.com/charcole/Dreamcast-PopnMusic/blob/main/Photos/IMG_4824.jpeg)
*Please excuse the kid mess in this photo :)*

## Software
The physical construction was done a couple years ago but I put off doing the software. The Maple (Dreamcast controller) bus is quite a fast bus so a bit tricky to do with a low end microcontroller (although can be done). When the Raspbery Pi Pico was announced the Programmable IO seemed like it would be perfect for implementing this which spurred me into action to get this finished off. As the RP2040 (chip in the Pico) is fairly new I thought I'd write up my first impressions.

### Programmable IO (PIO)
I started off with using the PIO to send Maple packets with the start and end sync pulses. This wasn't really needed as sending is quite straight forward and could easily be done with bit banging but I was keen to get to grips with the PIO with a simple case first. Working with more than one pin was a bit clunky (having to use side channels) but it was fairly simple to write. Sending became very simple as it's just a case of starting a DMA to the PIO. Was really fire and forget once working which is really cool.

Recieving, the bit I was excited for using PIO for, turned out to be a harder problem. The Maple bus (that Dreamcast controllers use to communicate with the console) is a two wire protocol (http://mc.pp.se/dc/maplewire.html) where the two wires take turns at being the clock. This makes it challenging to work with in the PIO as...
* You can select a pin to branch on but only one per PIO state machine (but Maple needs two)
* You can read inputs to a register but with no way of masking (that I can see) so is pretty useless :(
So you are limited pretty much to only using the wait instructions to watch certain pins. This would work to some extent but would be error prone if it got out of sync and there'd be no way of detecting the end of packet condition (without using the CPU to detect and reset the PIO but that defeats some of the point of using the PIO in the first place).

It all became a bit like a Zachtronics puzzle (which admittedly was fun). I did come up with an elegant way in the end but wasn't doing the decoding in the PIO which I was disappointed about. The solution I came up with was to have two state machines waiting on either pin changing and signalling another state machine to shift in the current state of both pins. This means it clocked in data only when there was a change on either of the pins. We can pack four transitions into a byte before pushing that out the FIFO. Limiting the number of transitions means we can still detect there's been an end of packet sequence even when there's some data left in the shift register waiting to be pushed.

Overall I was slightly disappointed with the PIO. I feel it would have been much more useful for input from multiple pins if there was just some masking of the input. Without a mask `mov x, pins` can't be used to compare via the `jmp x!=y` instruction because `x` ends up with a bunch of noise from all the other pins you aren't interested in.

*Disclaimer: This is first time I've used the PIO so could be missing something. The datasheet is a bit unclear about whether pins with lower indexes get rotated in or not. Could be a way of getting some masking if you use certain pins*

### DMA

The DMA engine seems really cool but has some quirks. For transmitting, like the PIO, it works great and the documentation/libraries are really good (IMO). However for recieving I ran into some more problems. I really just wanted an endless ring buffer that'd keep recieving and the CPU could consume the data as it came in.

The first hurdle was you can't have an endlessly running DMA. This isn't a huge issue as an interrupt can restart it fairly quickly (although later on I did question if I wasn't sometimes missing data). The second issue was with the looping. Although you can tell the DMA engine to only update certain bits of the address this didn't seem to work perfectly when I was trying to read the write address back on the CPU to know how much of the ring buffer to consume. Although I had a buffer 0x8000-0x8FFF for example, reading the write address on the CPU would rarely return an errant 0x7FFF. This caused me a fair bit of confusion as my loop was just reading until the current read pointer matched the DMA write pointer, which meant it could blast right by in rare occurances due to this glitch reading garbage old data.

I'm sure it was problems with my own code but I couldn't get this ring buffer DMA to work reliably. Combined with needing to speed up the communication (due to the Dreamcast sometimes sending super rapid requests when using the DC-X disc to load games from different regions) I decided to just ignore the DMA for recieving and use the second core in the end.

### Second core

To split up the work I decided to use the second core to just read directly from the reciever FIFO and do the Maple bus decoding into a ring buffer while the other core could process the packets and send the replies. The second core was very simple to get working and the mailbox system for communicating between the two was very easy to use.

I knew there might not be much time to process data (maybe as little as half a microsecond for each byte of input) to keep up with the receiving PIO so I used the large amount of RAM to build a table (20Kb) of precalculated responses so a byte of transitions could be decoded at a time with just a simple lookup. I was a bit shocked to discover this code couldn't keep up at first but after a bit of head scratching I realized the problem. Code is stored in flash and paged into a cache in RAM as needed. This is a slow process and if my time critical function wasn't paged in then by the time it was fetched from flash the input FIFO would be full and data would have been missed. The magic incantation I needed was to declare my function as `__not_in_flash_func(core1_entry)(void)` to force it to always be in RAM. For then on it was smooth sailing.

### PWM

I used PWM for all the LEDs so I could fade them slowly after each button press. Again everything was well documented and the library really easy to use. Only slight wrinkle was as there are less PWM channels than GPIO pins so I had to be careful not to have two LEDs on pins that shared the same PWM channel. This was easily done by just making sure they were all consecutive numbered IO pins.

### Development Environment

Development environment was really nice once it was set up. I started off doing the lazy way of pulling in the Pico with the button held down then dragging in the UF2 file to program. This worked but boy was it annoying. I really wish it just had a reset button. Balancing a laptop on my knee while holding a button and re-plugging in a USB cable, dragging a file then having to quickly reconnect to the USB serial to see the output got old fast.

I was lucky enough to get three Raspberry Pi Pico's on launch day so I used the Picoprobe software to use one Pico as a USB debug/serial interface for the other. This worked quite nicely and is great to have a debugger on a microcontroller. I know this is fairly standard for ARM microcontrollers but it was the first time I'd personally had this set up. OpenOCD would often crash, the Picoprobe needing rebooting or the debugger would just getting confused and not restart the program properly but I was still impressed and was way better than any other environment I've had on a microcontroller. Compared to the lazy set up it was light years ahead. I'm sure kinks will get ironed out over time and I was using macOS which probably isn't as well tested as other platforms.

### Overall impressions of the Pico

I know I've bought up some problems but I actually really like the Pico. It's new so information seems a bit thin on the ground but hopefully that'll improve in time. The Programmable IO is limited in some annoying ways but for what it's good at it's brilliant. The documentation and libraries are all top notch as far as I'm concerned (as a hobbiest). I have used a lot of ESP32s recently and has been my go to microcontroller for quite some years now. I'm not sure the Pico can unseat it as my favourite microcontroller just yet but it's giving it a good run for the money. I'm really looking forward to where the Pico goes next.

## Physical Construction
The controller itself is made from many laser cut slices of 6mm MDF. The top layer is clear acrylic and underneath that are the graphics printed on normal paper (I used a poster printing company). The bottom layer is cork matting to make a nice non slip surface. All the layers are sandwiched together using hefty M8 bolts apart from the bottom couple layers which are glued to provide a space for the nuts. In all it's a very heavy and substatial feeling controller that should live up to a lot of punishment like the real arcade machine.

It was designed in Fusion 360 back when the free version allowed DXF export. Might not be too much use now but I've included the Fusion file in case you want to modify it and you can find some way still to export it. I've included all the exported SVG files from when I originally laser cut it. I believe the graphics were also done in Fusion and then edited in Inkscape.

Buttons are generic 100mm arcade buttons found on eBay or AliExpress. Be warned I used the shorter stemmed ones. If you can only get the longer ones then you might have to add more layers. It seems random whether you get LED and resistor or just an LED included in the button, not to mention different colours. I removed them all in any case and replaced them with a high brightness white LED with a 82ohm resistor for consistancy.

For strain relief on the cable I left room for the wire to curl around one of the bolts a couple times. As I didn't want the wire to be cut by the thread of the bolt I asked a friend to show me how to use the lathe at the local hackspace (where I also used the laser cutter) and we machined a small tube out of Delrin with an M8 thread on the inside (so it doesn't rotate). I suppose you could get similar results with a 3D printed spacer though if you don't have access to a friendly machinist.

I did leave a slot within the shell for the electronic but I ended up just putting the Pico in one of the button wells as there was more space for Dupont connectors (for easy servicability). I might do a PCB at some time in the future and try out the castellations for soldering the Pico as a module. 
