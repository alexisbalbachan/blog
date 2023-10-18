# Table of Contents
* [Motivation](#motivation)
* [Introduction](#introduction)
* [Physical Layer](#physical-layer)
   * [Link Cable Pinout](#link-cable-pinout)
   * [Signals](#signals)
   * [SPI](#spi)
     * [Concept](#concept)
     * [How Gameboys Implement SPI](#how-gameboys-implement-spi)
     * [Additional Restrictions in Pokemon Games](#additional-restrictions-in-pokemon-games)
* [Pokemon Trading Protocol](#pokemon-trading-protocol)
  * [Generation I](#generation-i)
  * [Generation II](#generation-ii)
  * [Time Capsule](#time-capsule)


## Motivation
I grew up in the midst of the pokemon craze in the 90s, in my school the series became popular first, and soon after everyone was bringing in their gameboys to play the Pokemon games on breaks.

Eventually my parents got me a brand new Gameboy Color alongside a Pokemon Red cartridge, so my quest for catching 'em all began! 

The most frustrating thing about those games was **trading**, not only did i have to find someone playing Pokemon blue and willing to trade with me, but that person also had to own a Link Cable.

Not many children at my school owned a Link Cable so completing my Pokedex was really **\*HARD\*** (My original save file is still alive btw!).

What about Mew? Well, a couple of years later i got a Gameshark which i used to capture it.

The really interesting thing about all of this is that the Gameshark cartridge came with a Gameboy to PC(Parallel? port) cable which is the closest thing i had that resembles a Link Cable:

 <p align="center"><img src="/docs/assets/images/s-l1600.png" alt="Gameshark Link to PC Cable" style="width:500px;"/></p>
 <p align="center"><em>Gameboy to PC cable</em></p>

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
5. One Gameboy will generate a clock signal (see SPI section) in this pin to synchronize the transmission. Both Gameboys will read from and write into their corresponding pins only when the clock signal reaches specific states.
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

 <p align="center"><img src="/docs/assets/images/input_signal2.png" alt="Interpretation2" style="width:500px;"/></p>

 Also, where does the transmission actually start? (and end?):
 
 <p align="center"><img src="/docs/assets/images/input_signal3.png" alt="Interpretation3" style="width:500px;"/></p>

 I that even a single transmission?:
 
 <p align="center"><img src="/docs/assets/images/input_signal4.png" alt="Interpretation4" style="width:500px;"/></p>

 <br>

SPI solves this problem by introducing a **shared clock** which indicates when to read from and write to the data lines.

<p align="center">A clock is just a signal that switches between HIGH and LOW at regular intervals:</p>

<p align="center"><img src="/docs/assets/images/clocks.png" alt="Clock examples" style="width:500px;"/></p>

Where does the clock signal come from? One of the peers generates it while the other one just reads from that line!

* The peer that sends the signal is named **MASTER** (can be also named **CONTROLLER** or **MAIN**).
* The one that only reads from the clock line is called **SLAVE** (or **PERIPHERAL**, or even **TARGET**).
* Data lines are also named after those roles:
  * The line where the master writes into and the slave reads from is called **M**aster **O**ut **S**lave **I**n (**MOSI**).
    * It can also be called **C**ontroller **O**ut **P**eripheral **I**n (**COPI**).
    * Or **C**ontroller **O**ut **T**arget **I**n (**COTI**).
  * The line where the slave writes into and the master reads from is called  **M**aster **I**n **S**lave **O**ut (**MISO**).
    * It can also be called **C**ontroller **I**n **P**eripheral **O**ut (**CIPO**).
    * Or **C**ontroller **I**n **T**arget **O**ut (**CITO**).
* There is also a fourth signal which indicates when the transmission starts and ends:
  * This line is controlled by the master aswell.
  * Its called **S**lave **S**elect (**SS**), **C**hip **S**elect (**CS**), **C**hip **E**nable (**CE**).
  * NOTE: SPI can support multiple slave nodes, and each one needs to have a separate **SS** line to the master (other lines are shared). They will only communicate when their **SS** line is active.
* Both Master and Slave send and receive data at the same time.
 
<p align="center">Here's an example of what SPI transmissions looks like (MOSI: 1011, MISO: 0110)</p>

<p align="center"><img src="/docs/assets/images/SPI_transmission.png" alt="Clock examples" style="width:500px;"/></p>

Note that **in this example** MOSI and MISO signals will only change when the clock signal is LOW.

* SPI operates in 4 different modes which are basically a set of rules that both master and slave have to agree upon beforehand.
* There are 2 parameters to consider:
  * Does the clock stay idle at HIGH or at LOW? This is called **C**lock **POL**arity (**CPOL**).
  * Is data read when the clock changes to LOW or to HIGH? and in that same line, is data written when the clock changes to LOW or to HIGH? This is called **C**lock **Pha**se (**CPHA**).
* Read/write should be done on **clock edges** which is the moment when the clock changes values, a rising edge goes from LOW to HIGH and a falling edge goes from HIGH to LOW.
* Its more important to write on time than to read on time, you can actually read your data whenever you want before the next clock edge.

<br>

<div align="center">

Here is a table briefly describing the 4 SPI modes:
  
| SPI Mode | Clock Polarity when idle | When to read/write (edges) |
|----------|--------------------------|----------------------------|
| 0        |           LOW            | R on RISING / W on FALLING |
| 1        |           LOW            | R on FALLING / W on RISING |
| 2        |           HIGH           | R on FALLING / W on RISING |
| 3        |           HIGH           | R on RISING / W on FALLING |
  
</div>

<br>

Most SPI implementations will only work with message lengths that are multiple of 8 bits (so the smallest unit that can be sent is 1 byte)

There is an additional thing to consider and that is the bit order of the data being read/written:
* If you wanted to send any number, for example 42 whose binary representation is 00101010 then you could write it from right to left or from left to right.
  * Writing from left to right (0->0->1->0->1->0->1->0) means that we're writing from its most significant bit onward (**MSB**).
  * Writing from right to left (0->1->0->1->0->1->0->0) means that we're writing from its less significant bit onward (**LSB**).
* Whatever order is chosen, both ends must respect it when writing and reading from the data lines.
  * In our example if we read 42 in a different order (LSB instead of MSB or MSB instead of LSB) then we'll end up with a flipped number: 01010100 (84 in decimal).

<br>
<br>

#### How Gameboys Implement SPI

Here i'll describe some key aspects of the SPI implementation used by the Gameboys.

A detailed description about the implementation can be found [HERE](https://gbdev.io/pandocs/Serial_Data_Transfer_%28Link_Cable%29.html) (Pandocs) and more importantly [HERE](https://ia803208.us.archive.org/9/items/GameBoyProgManVer1.1/GameBoyProgManVer1.1.pdf#%5B%7B%22num%22%3A158%2C%22gen%22%3A0%7D%2C%7B%22name%22%3A%22FitH%22%7D%2C769%5D) (Gameboy Programming Manual Chapter 1 Section 2.4).

 <p align="center"><img src="/docs/assets/images/SIO_block_diagram.png" alt="Serial communication block diagram" style="width:500px;"/></p>
 <p align="center"><em>Serial communication block diagram (Gameboy Programming Manual)</em></p>

<br>

* Gameboys use **SPI mode 3** when communicating through the link cable, so the clock signal stays HIGH when idle, data is written on falling edges and is read on rising edges.
* Data is transfered byte by byte, that means that once 8 bits are transfered, an interrupt will trigger to signal the software that a byte has arrived/has been sent.
* Bytes are written/read from their **M**ost **S**ignificant **B**it (**MSB**) onwards, i.e. from left to right.
* **THERE IS NO SS/CS LINE**: The master will just generate a clock signal and the slave will read/write from/into data lines as soon as it detects the corresponding clock edges.
* This should be obvious but got me confused at first: The clock signal will only exist when data is being transferred, for some reason i thought that it was always active as long as the connection was established.
* The original Gameboy can only generate a 8192Hz clock (1 cycle/bit every 122μs or every 0.000122 seconds!).
* Gameboy Color can generate clocks at 4 different frequencies: 8192Hz, 16384Hz, 262144Hz, and 524288Hz.
* Slave Gameboys have to adapt to any incoming clock signal, they support absurdly low speeds (even 1 cycle per month!) and up to 500KHz (original GB).
  
* **Master/Slave Negotiation**:
  * Gameboys first poll the clock line, if they find a clock signal then they'll act as a slave. If no signal is present then they'll start sending their own clock signal (thus becoming master).
  * This means that the first Gameboy that tries to initiate a connection will probably become master.
 

<br>

Things to keep in mind:

* Slave nodes **cannot** send anything on their own, they have to wait for the master's clock signal first (which happens only when the master wants to transfer something themselves).
* Similarly slaves are unable to signal if they're ready for the transfer or not, it will occur anyway and the master will read whatever is in its input signal.
* This is why master nodes need to be considerate and give some time between each transfer to make sure that the slave is ready for it!
* Transfers are simultaneous, but the slave may have to read something from its master before knowing **what** to send, so in many implementations slaves are 1 step behind their master:
  * (MASTER <-> SLAVE):  (1) REQUEST1 <-> NOTHING | (2) REQUEST2 <-> RESPONSE1 | (3) REQUEST3 <-> RESPONSE2 | etc.

<br>
<br>

#### Additional Restrictions in Pokemon Games

In the previous section i described the SPI capabilities of the Gameboy Hardware, **HOW** they'll be used depends on the software (i.e. the games):

* Pokemon Games will use the lowest clock speed for transmissions (8192Hz). I'm assuming that this is to support trades between any hardware (original GB, GBP, and GBC).
* There seems to be a limit on how SLOW a clock can be, so slaves will timeout if no clock edge is detected after a certain amount of time.
* I'm not sure if slave pokemon games would support much higher clock speeds than the standard rate (haven't tested it yet).
* It could be possible to change the clock rate at any time as long as it does no exceed the allowed limits.

<br><br><br>

## Pokemon Trading Protocol

All the information provided in previous sections was tiresome to come by, as it's scattered all over the internet. But the important thing is that **it exists**!

As mentioned before, i could only find **one** single resource that touches on this topic ([Check it out!](http://www.adanscotney.com/2014/01/spoofing-pokemon-trades-with-stellaris.html)). It was crucial for my project and for my understanding of the protocol, however it came with several issues:
* The provided protocol description is.. vague to say the least (but a good place to start!)
* Both the protocol description and the provided PoC **are incomplete** (Barely enough to trade pokemon in the simplest scenarios).
* The code heavily relies on mirroring data to work (which may break some trades).
* It also expects the user to provide a valid buffer (which represents their pokemon data). The buffer itself has to follow certain rules not mentioned elsewhere!
* Some code may be missing (the demo video shows 2 pokemon being traded, but the code itself only reaches the trade screen).
* It only focuses on **Gen I**.

There are other resources available that implement the trading protocol but unfortunately it seems that all of them are heavily based on the one mentioned above (some even use the exact same code).

The actual trading algorithms implemented in the games are available [HERE (Pokered: GEN I)](https://github.com/pret/pokered/blob/b302e93674f376f2881cbd931a698345ad27bec3/engine/link/cable_club.asm) and [HERE (Pokegold: GEN II)](https://github.com/pret/pokegold/blob/70f883dc8670c95f411a9bbfa6ff13c44c027632/engine/link/link.asm). Both projects were really helpful but the code is written in pure Gameboy assembly (similar to Z80) with little to no comments.

I'll describe the trading protocol for generation I first, then include how generation II differs from it and what was added (both generations have a lot in common so its important to read both sections).

### Generation I



### Generation II

### Time Capsule
  

[1]: https://hackaday.io/project/160329-blinky-for-game-boy/log/150762-game-link-cable-and-connector-pinout
[2]: https://www.insidegadgets.com/2018/12/09/making-the-gameboy-link-cable-wireless-packet-based/
[3]: https://dhole.github.io/post/gameboy_serial_1/
[4]: https://en.wikipedia.org/wiki/Serial_Peripheral_Interface
[5]: https://www.analog.com/en/analog-dialogue/articles/introduction-to-spi-interface.html
[6]: https://learn.sparkfun.com/tutorials/serial-peripheral-interface-spi/all
[7]: https://www.circuitbasics.com/basics-of-the-spi-communication-protocol/
[8]: https://wiki.dfrobot.com/how_the_spi_protocol_works
