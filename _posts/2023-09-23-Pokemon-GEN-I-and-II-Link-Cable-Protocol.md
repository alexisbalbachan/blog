# Table of Contents
* [Motivation](#motivation)
* [Introduction](#introduction)
* [Physical Layer](#physical-layer)
   * [Link Cable Pinout](#link-cable-pinout)
   * [Signals](#signals)
   * [SPI](#spi)


## Motivation
I grew up in the midst of the pokemon craze in the 90s, in my school the series became popular first, and soon after everyone was bringing in their gameboys to play the Pokemon games on breaks.

Eventually my parents got me a brand new Gameboy Color alongside a Pokemon Red cartridge, so my quest for catching 'em all began! 

The most frustrating thing about those games was **trading**, not only did i have to find someone playing Pokemon blue and willing to trade with me, but that person also had to own a Link Cable.

Not many children at my school owned a Link Cable so completing my Pokedex was really **\*HARD\*** (My original save file is still alive btw!).

What about Mew? Well, a couple of years later i got a Gameshark which i used to capture it.

The really interesting thing about all of this is that the Gameshark cartridge came with a Gameboy to PC(Parallel? port) cable which is the closest thing i had that resembles a Link Cable:

 <p align="center"><img src="/docs/assets/images/s-l1600.png" alt="Gameshark Link to PC Cable" style="width:500px;"/></p>

<!--![Gameshark Link to PC Cable](/docs/assets/images/s-l1600.png)-->

That cable was used to upload/download cheats into the Gameshark, also to update its firmware. So of course i had to try it... Which ended up bricking my precious Gameshark. Now with a useless cartrige, the cable ended up collecting dust for decades.

About 5 years ago, while tinkering with a raspberry pi i suddenly remembered **the cable**. Would it be possible to trade Pokemon between a Gameboy and a PC?

I started searching for any existing information about this subject and it was mainly all bad news:

(Note: I was and still am a complete noob at this kind of stuff)

* Gameboy to PC:
    * To this day i'm still not sure **HOW** this cable was originally used, is it serial? Is it parallel?
    * If serial: It seems that PC serial ports communicate using the RS232 protocol which works with voltage levels way beyond of the 0v/+5v signals that the Gameboy can handle (I've seen the name MAX-232 thrown around alot for adapting the signals) -> *Overwhelmed*.
    * If parallel: Voltage levels match, so far so good... But my PC doesn't have a parallel port anymore, even if i had one.. How would i control individual pins (either in Windows or Linux)? -> *Overwhelmed*.
* Gameboy to Raspberry Pi:
    * Already knew how to read from / write to individual pins, great!
    * **GPIO pins work with 3.3V and are NOT 5V tolerant** (DO NOT CONNECT RPI PINS TO A GAMEBOY DIRECTLY).
    * Intro to logic level converters before even connecting a single cable -> *Overwhelmed*.

<br>

The only thing that kept me going was [THIS BLOGPOST](http://www.adanscotney.com/2014/01/spoofing-pokemon-trades-with-stellaris.html) (Thank you [Adan Scotney](http://www.adanscotney.com)!) in which **someone** accomplished the exact same thing i was trying to do! However he used a.. "Stellaris Launchpad" which i had NO IDEA what it was (still don't know).

So i knew **it was possible**, it wasn't extremely difficult, and had all the information required to implement it on my own (or so i thought..). The only thing i didn't have at the time was the hardware which i had no intention of buying and just kept the idea in the back of my mind while working on other projects..


Fast forward to **2023**: I found out that Arduino boards were REALLY cheap (compared to a RPI), and bought myself an Arduino Nano, a bunch of sensors, and started delving into the world of microcontrollers. Did some basic examples, then implemented communication protocols with different sensors until i felt confident for my next task: Trade pokemon between a PC and a Gameboy using the Arduino as an intermediary. \[**SPOILER: I DID IT!**\]

 <p align="center"><img src="/docs/assets/images/PoC.png" alt="Pokemon Trade GUI" style="width:500px;"/></p>
<!-- ![Pokemon Trade GUI](/docs/assets/images/PoC.png) -->

## Introduction
The main goal of this post is to explain how trading works in pokemon red/blue/yellow and gold/silver/crystal in as much detail as possible, from the physical layer to the actual protocol implemented by those games. 

I'll also include the misconceptions and problems encountered along the way while implementing said protocol (many of the problems will look trivial to you!)


## Physical Layer

### Link Cable Pinout
The cable pins have already been described many times ([1], [2], [3])

                                                                  ___________
                                                                 |  2  4  6  |
                                                                  \_1__3__5_/

<div align="center">

| #           | Name        | Description |
| ----------- | ----------- | ----------- |
| 1           | VCC         | +5V         |
| 2           | SOUT        | Serial Out  | 
| 3           | SIN         | Serial In   | 
| 4           | Not Used    | Not Used    | 
| 5           | SCK         | Serial Clock| 
| 6           | GND         | Ground      | 

</div>

1. Gameboys are able to provide 5V through this pin, some (unofficial?) accessories use it (E.G: night lights), its not used when connected to another Gameboy.
2. Data is sent through this pin bit by bit. It is connected to SIN on the other side of the cable.
3. Data is received through this pin bit by bit. It is connected to SOUT on the other side of the cable.
4. Not used/Usage unknown/Haven't found any additional reference to this pin.
5. One Gameboy will generate a clock signal (see SPI section) in this pin to synchronize the exchanges. Both Gameboys will read from and write into their corresponding pins only when the clock signal reaches specific states.
6. Ground reference pin so both Gameboys share a common ground (EXTREMELY IMPORTANT **\***).

\* My first mistake was ignoring the ground pin, i was just starting and tried to read what was being sent on each pin.. I remember reading the exact opposite of what was expected (E.G: Expected 101010 , got 010101 instead). I lost so much time trying to figure that out! The solution was connecting pin 6 to any of my arduino's GND pins.

### Signals

* Voltage levels on each pin should be either 0V (representing 0/LOW) or 5V (representing 1/HIGH).
* I haven't tested this, but lower voltages may still work (e.g 3V or 4V could also be considered as HIGH). Just make sure that whatever communicates with a Gameboy can tolerate 5V on its pins.
* I would't recommend using voltages higher than 5V as you could damage your Gameboy.

<br>

* Both SOUT and SIN normally stay at LOW (0V) when idle (it shouldn't really matter)
* SCK stays HIGH when idle, a pullup resistor will ensure that 5V are read even when there's nothing connected on the other end of the cable.
* Using a pulldown resistor on SCK should probably work, we're only looking to read a consistent value from that pin when the other side isn't transmitting anything (or isn't even connected), otherwise the value will fluctuate randomly and we may confuse those fluctuations with an actual clock signal!

### SPI

#### Concept

At this layer (physical) we'll only be concerned with transmitting bits and bytes, their actual meaning will be dealt with in higher layers.

SPI (**S**erial **P**eripheral **I**nterface) is a widely used communication protocol, its amazingly well documented ([4], [5], [6], [7], [8]) and is (in my opinion) quite easy to understand. 

At its most basic level it has 2 lines used for sending and receiving data, one line for syncronization between the 2 peers, and one line for signaling the beginning/end of the trasmission.

Why synchronization? Well, the peers need to know **when** to read from their input line and **when** to write into their output line, consider the following scenario:

 <p align="center"> Assuming LOW is 0 and HIGH is 1.. What binary number is being transmitted? </p>

 <p align="center"><img src="/docs/assets/images/input_signal.png" alt="Mystery signal" style="width:500px;"/></p>


 
 The safest bet would be 000101010:
 
  <p align="center"><img src="/docs/assets/images/input_signal1.png" alt="Interpretation1" style="width:500px;"/></p>

  But, there are other interpretations that are equally valid:

   <p align="center"><img src="/docs/assets/images/input_signal2.png" alt="Interpretation1" style="width:500px;"/></p>

[1]: https://hackaday.io/project/160329-blinky-for-game-boy/log/150762-game-link-cable-and-connector-pinout
[2]: https://www.insidegadgets.com/2018/12/09/making-the-gameboy-link-cable-wireless-packet-based/
[3]: https://dhole.github.io/post/gameboy_serial_1/
[4]: https://en.wikipedia.org/wiki/Serial_Peripheral_Interface
[5]: https://www.analog.com/en/analog-dialogue/articles/introduction-to-spi-interface.html
[6]: https://learn.sparkfun.com/tutorials/serial-peripheral-interface-spi/all
[7]: https://www.circuitbasics.com/basics-of-the-spi-communication-protocol/
[8]: https://wiki.dfrobot.com/how_the_spi_protocol_works
