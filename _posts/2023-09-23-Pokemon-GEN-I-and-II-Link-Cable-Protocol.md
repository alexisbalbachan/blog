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
    * [Handshake](#handshake)
    * [Players Ready](#players-ready)
    * [Room Selection](#room-selection)
    * [Players Ready for Trade](#players-ready-for-trade)
    * [Seed Exchange](#seed-exchange)
    * [Party Information Exchange](#party-information-exchange)
    * [Data Structure](#data-structure)
      * [Trainer Name](#trainer-name)
      * [Party Size](#party-size)
      * [Pokemon Id List](#pokemon-id-list)
      * [Pokemon Structure (x6)](#pokemon-structure-x6)
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

<div align="center">

<hr>
<br>
  
#### Handshake

The first Gameboy to initiate the connection (by talking to the Cable Club NPC at the Pokemon Center) will probe the SCK line and won't find any clock signal there, thus becoming **master**.
It will repeatedly send **0x01** looking for a response on the other side, the slave gameboy must always reply with **0x02** (everytime it receives 0x01). 

Note that the slave gameboy won't reply to the first 0x01 because it can only reply to what it has already been received, (the master->slave and slave->master transmission happens at the same time!) so the first message will always be replied with 0x00:

| #MSG | MASTER | SLAVE              |
|------|--------|--------------------|
|1     | 0x01   | 0x00               |
|2     | 0x01   | REPLY for #1 HERE  |

On my tests, the master kept trying to establish a connection approximately 89 times (i.e. sends 0x01 89 times with no response on the other side):

| #MSG | MASTER | SLAVE |
|------|--------|-------|
|1     | 0x01   | 0x00  |
|2     | 0x01   | 0x00  |
|3     | 0x01   | 0x00  |
|4     | 0x01   | 0x00  |
|5     | 0x01   | 0x00  |
|......|........|.......|
|89    | 0x01   | 0x00  |
|90    | ABORT  | ABORT |

Only one 0x02 response is needed for the master to accept the handshake, further responses are ignored, they can be of any value (i'd recommend to keep responding with 0x02):

Expected successful handshake:

| #MSG | MASTER           | SLAVE                 |
|------|------------------|-----------------------|
|1     | 0x01             | 0x00                  |
|2     | 0x01             | 0x02 (Responds to #1) |
|3     | ACKNOWLEDGED!    | 0x02 (Responds to #2) |
|4     | .................| ......................|

Abnormal successful handshake (still valid):

| #MSG | MASTER           | SLAVE                 |
|------|------------------|-----------------------|
|1     | 0x01             | 0x00                  |
|2     | 0x01             | 0x02 (Responds to #1) |
|3     | ACKNOWLEDGED!    | **0x99** (Responds to #2) |
|4     | .................| ......................|

Once a handshake is accepted by the master, it acknowledges it by sending **0x00** (a very bad choice for an acknowledge, it is the same as not sending anything!). The slave must also reply with **0x00**, in my tests the acknowledge happens twice (both master and slave send 0x00 two times in a row)

**A complete handshake looks like this:**

| #MSG | MASTER | SLAVE |
|------|--------|-------|
|1     | 0x01   | 0x00  |
|2     | 0x01   | 0x02  |
|3     | 0x00   | 0x02  |
|4     | 0x00   | 0x00  |
|5     | 0x00   | 0x00  |

After a handshake both players are asked to save the game.

<hr>
<br>

#### Players Ready

The master won't send anything else until the player saves the game, similarly the slave will always reply with 0x00 until its player also saves their game.

Once the master saved its game it will repeatedly send **0x60** (to signal that it's ready) until it receives **0x60** as well. It will abort and disconnect after several seconds of not receiving that same value! (it sends 0x00 before disconnecting)

| #MSG   | MASTER      | SLAVE |
|--------|-------------|-------|
|X       | 0x60        | 0x00  |
|X+1     | 0x60        | 0x00  |
|X+2     | 0x60        | 0x00  |
|X+3     | 0x60        | 0x00  |
|....... | ........... | ..... |
|X+7679  | 0x60        | 0x00  |
|X+7680  | ABORT(0x00) | 0x00  |

After saving its game the slave will also abort after some time without any messages coming from the master, but because it cannot send anything on its own it will just disconnect:

| #MSG   | MASTER      | SLAVE                 |
|--------|-------------|-----------------------|
|X       | FIRST 0x60  | ALREADY DISCONNECTED  |

<br><br>

If both players are ready on that time window then the slave will be responding with 0x60 everytime it receives that value. This can happen multiple times until the master acknowledges it (sending **0x00**, the default acknowledge message). The slave will also reply the acknowledge message with its own acknowledge (also **0x00**):

| #MSG | MASTER       | SLAVE       |
|------|--------------|-------------|
|X     | 0x60 (FIRST) | 0x00        |
|X+1   | 0x60         | 0x60 (FIRST)|
|X+2   | 0x60         | 0x60        |
|X+3   | 0x60         | 0x60        |
|X+4   | 0x60         | 0x60        |
|X+5   | 0x60         | 0x60        |
|..... | ............ | ........... |
|X+13  | 0x60 (LAST)  | 0x60        |
|X+14  | 0x00 (ACK)   | 0x60 (LAST) |
|X+15  | 0x00         | 0x00 (ACK)  |
|X+16  | 0x00         | 0x00        |
|X+17  | 0x00         | 0x00        |
|..... | ............ | ........... |
|X+24  | 0x00         | 0x00        |
|X+25  | MENU MSG     | 0x00        |

<hr><br>

#### Room Selection

This is where players select what they want to do: **Trade, Battle (Colosseum), or Cancel**. They can also exit the menu (by pressing **B**) at any time.

Both gameboys will continuously send which menu is currently being selected, the messages are sent twice and then there's one acknowledge before repeating the loop:


| #MSG    | MASTER       | SLAVE       |
|---------|--------------|-------------|
|X        | MENU MSG #1  | 0x00        |
|X+1      | MENU MSG #2  | MENU MSG #1 |
|X+2      | 0x00         | MENU MSG #2 |
|X+3      | MENU MSG #1  | 0x00        |
|X+4      | MENU MSG #2  | MENU MSG #1 |
|X+5      | 0x00         | MENU MSG #2 |
|X+6      | MENU MSG #1  | 0x00        |
|X+7      | MENU MSG #2  | MENU MSG #1 |
|X+8      | 0x00         | MENU MSG #2 |
|INFINITY | ............ | ........... |


<br>

Menu messages have the following format:

<table>
  <tr>
    <td>HEX:</td>
    <td colspan="4">0xD</td>
    <td colspan="4">0xY</td>
  </tr>
  <tr>
    <td>Binary:</td>
    <td>1</td>
    <td>1</td>
    <td>0</td>
    <td>1</td>
    <td>y</td>
    <td>y</td>
    <td>y</td>
    <td>y</td>
  </tr>
  <tr>
    <td>Meaning:</td>
    <td colspan="4">PREFIX</td>
    <td>B</td>
    <td>A</td>
    <td colspan="2">#M</td>
  </tr>
</table>

<br>

The highest 4 bits of menu messages are always **0xD**.

The lowest 4 bits are the ones describing the action:

| Symbol| Description                   |
|-------|-------------------------------|
| B     | 1 if Player pressed "B"       |
| A     | 1 if Player pressed "A"       |
| #M    | Menu item number (0, 1, or 2) |

</div>

* #M = 0: Trade, #M = 1: Colosseum, #M = 2: Cancel
* Colosseum takes priority over Cancel so #M = 3 (1 and 2 at the same time) is considered Colosseum (it shouldn't be a valid message anyways!).
* If both A and B are 0 then #M indicates which item is currently being highlighted.
* If B is 1 then the player quit the selection menu while #M was being highlighted, it is considered as a cancel. Both gameboys disconnect.
* B takes priority over A, so if both A and B are 1 then it will also be treated as a cancel (This type of message shouldn't exist under normal circumnstances).
* If A is 1 then the player selected #M in the menu.
  * If A is 1 and #M is 2 then the player chose "Cancel" both gameboys will abort and disconnect right now.
* **The first player that selects (or cancels) something will pick that option for both of them!**
* The selection will be made effective once the master ends an iteration (after sending an acknowledge(0x00)). The selection loop ends there and the master stops communicating (for now).
 
<br>

<div align="center">
  
Here's how it looks like when players select and choose the trade option in the menu (slave chose first!):


| #MSG    | MASTER | SLAVE|
|---------|--------|------|
|X        | 0xD0   | 0x00 |
|X+1      | 0xD0   | 0xD0 |
|X+2      | 0x00   | 0xD0 |
|X+3      | 0xD0   | 0x00 |
|X+4      | 0xD0   | **0xD4** |
|X+5      | 0x00   | **0xD4** |
|X+6      | DONE   | DONE |

<hr><br>

#### Players Ready for Trade

This step is very similar to [Players Ready](#players-ready)

</div>

**Both players are now in the trade room!** Here, they're expected to press "A" facing the trading machine.
Any other action (walking around) isn't communicated to the other peer! The cable can even be unplugged and nothing will break.

When the master is ready (presses "A"), it will first send a single **0xFE** which means that it doesn't have any remaining data to send (i'm not sure **WHY** it would have something else to send at this point). Followed by an endless stream of **0x60** until it receives several 0x60 from the slave as well. **There isn't a timeout here**. The master will be stuck waiting until the Gameboy is turned off.

The same happens with the slave when it iteracts with the machine: It will wait for the master to communicate (and respond with a single 0xFE at first followed by 0x60 on subsequent transmissions). If the master never communicates then it will be stuck forever until the gameboy is turned off.

After receiving many successful responses, the master will send a stream of 0x00 before continuing.

<div align="center">

Here's what it looks like when both gameboys are ready (Master was ready first):

| #MSG    | MASTER | SLAVE|
|---------|--------|------|
|X        | 0xFE   | 0x00 |
|X+1      | 0x60   | 0x00 |
|X+2      | 0x60   | 0x00 |
|X+3      | 0x60   | 0x00 |
| ....... | ...... | .... |
|X+5000   | 0x60   | 0xFE |
|X+5001   | 0x60   | 0x60 |
|X+5002   | 0x60   | 0x60 |
|X+5003   | 0x60   | 0x60 |
|X+5004   | 0x60   | 0x60 |
| ....... | ...... | .... |
|x+5014   | 0x00   | 0x60 |
|x+5015   | 0x00   | 0x00 |
|x+5016   | 0x00   | 0x00 |
|x+5017   | 0x00   | 0x00 |
| ....... | ...... | .... |
|x+5022   | 0x00   | 0x00 |
|x+5023   | READY  | READY|

<hr><br>

#### Seed Exchange

</div>

A random seed is used when battling in order to calculate things like critical hits, misses, etc. However, they are also exchanged when trading even though they have no use there!
Master and slave will exchange 10 random numbers between 1 and 252 (0xFC)

First the master will send **0xFD** (which is used as a **preamble** before a block of data is exchanged) repeatedly until it receives the same answer from the slave (it needs multiple successful answers before continuing, probably as a sync mechanism).

If the slave never responds with 0xFD, then the master will be stuck waiting forever.

Seed values go from 1 to 0xFC, so the first number in that range after the preamble stream is considered as the first seed value, and the subsequent 9 values are expected to be the remaining part of the seed (a total of 10 bytes).

I haven't tested what happens when sending 0xFD/0xFE/0xFF as part of the seed (which shouldn't happen).

<div align="center">

A normal seed exchange (Master seed: A0-A9, Slave seed: B0-B9)


| #MSG    | MASTER | SLAVE|
|---------|--------|------|
|X        | 0xFD   | 0x00 |
|X+1      | 0xFD   | 0xFD |
|X+2      | 0xFD   | 0xFD |
|X+3      | 0xFD   | 0xFD |
|X+4      | 0xFD   | 0xFD |
|X+5      | 0xFD   | 0xFD |
|X+6      | 0xFD   | 0xFD |
|X+7      | 0xFD   | 0xFD |
|X+8      | 0xFD   | 0xFD |
|X+9      | 0xFD   | 0xFD |
|X+10     | 0xA0   | 0xFD |
|X+11     | 0xA1   | 0xB0 |
|X+12     | 0xA2   | 0xB1 |
|X+13     | 0xA3   | 0xB2 |
|X+14     | 0xA4   | 0xB3 |
|X+15     | 0xA5   | 0xB4 |
|X+16     | 0xA6   | 0xB5 |
|X+17     | 0xA7   | 0xB6 |
|X+18     | 0xA8   | 0xB7 |
|X+19     | 0xA9   | 0xB8 |
|X+20     | 0xA9   | 0xB8 |
|X+21     | DONE   | 0xB9 |
|X+22     | DONE   | DONE |

<hr><br>

#### Party Information Exchange


</div>

This step is almost exactly the same as the seed exchange but with a bigger payload. Immediately after exchanging seeds, the master will start sending several **0xFD** as a preamble, and again, the slave must also reply with 0xFD or else the other side will be stuck waiting forever.

The exchange will start after the first non 0xFD byte appears and a total of **415 bytes** will be exchanged.

There's only one restriction: **The payload must not contain 0xFE**. 0xFE means that there is no more data to be exchanged, i'm not exactly sure how it works internally, but once the exchange ends the other side will just ignore everything after that byte (it's like an end of buffer marker or something like that).

What happens when 0xFE is part of the payload? It needs to be patched (see [Patch Section](#patch-section)). To keep it simple for now: **Each 0xFE byte is replaced with 0xFF before being sent**.

<div align="center">

A normal exchange (Preamble + 415 bytes):


| #MSG    | MASTER | SLAVE|
|---------|--------|------|
|X-10     | 0xFD   | 0x00 |
|X-9      | 0xFD   | 0xFD |
|X-8      | 0xFD   | 0xFD |
| ....... | ...... | .... |
|X        | 0x80   | 0xFD |
|X+1      | 0x88   | 0x82 |
|X+2      | 0x88   | 0x88 |
|X+3      | 0x95   | 0x90 |
| ....... | ...... | .... |
|X+414    | 0x50   | 0x80 |
|X+415    | END    | 0x50 |
|X+416    | END    | END  |

</div>


<div align="center">

<hr> <br>

#### Data Structure

</div>


* This is not part of the protocol per se, but it's important to understand **WHAT** is being exchanged in those 415 bytes as it affects the final result.

* There are many strings within that payload, they use **custom encodings (not ASCII!)**, which depends on the game's language (that's why trading between games of different languages will break their save files!).

* Those encodings are pretty well documented here: [Bulbapedia-> Character Encoding (Generation I)](https://bulbapedia.bulbagarden.net/wiki/Character_encoding_(Generation_I)).



<div align="center">

A few examples:

| Character | English Encoding | Japanese Encoding |
|-----------|------------------|-------------------|
|     A     |       0x80       |        0x60       |
|     B     |       0x81       |        0x61       |
|     a     |       0xA0       |   DOES NOT EXIST! |
|     5     |       0xFB       |        0xFB       |
|     ポ    |  DOES NOT EXIST! |        0x43       |

<br><br>

##### Trainer Name

</div>

* Bytes [0:10] (first 11 Bytes) correspond with the trainer's name which is displayed on the trade menu.
* The name itself is actually shorter, up to 10 bytes long, the extra byte is reserved for the string terminator (**0x50** in any encoding).
* Characters after 0x50 are ignored, they're usually **0x00**.
* A 3 byte name will have 0x50 at its fourth byte (followed by 0x00 afterwards), whereas a 10 byte name will have 0x50 in the (reserved) eleventh byte.
* I haven't tested what could happen if no string terminator is present or if we sent a 11 character string.
* **THIS NAME CANNOT BE PATCHED**. It cannot contain 0xFE (**"8"**), this can't happen without cheats/glitches because the in-game's keyboard doesn't have any numbers! This means you can't give a name with numbers to your player.
* **IT CAN'T START WITH 0xFD ("7")** as it will be interpreted as part of the preamble, messing up everything that comes after! Same as above, names don't usually contain numbers without cheats/glitches.
* **IT CAN'T START WITH NULL CHARACTERS (0x00)**. Also an invalid name because not only it's empty, but also its first character isn't 0x50! Anyways, null characters will be interpreted as acknowledges and therefore skipped.

<br><br>
<div align="center">

##### Party Size

</div>

* Byte [11]. This is only one byte with a value between 1 and 6. It specifies how many pokemon are there in the trainer's party.
* It's the only way of telling the other side that your party is smaller than 6, you have to exchange 415 bytes no matter how many pokemon you have! By specifying a smaller party size, the other peer will ignore the extra bytes (corresponding to non existing pokemon).
* [Glitch] if its bigger than expected (e.g: 6 when we only have 3 pokemon), the empty/random data corresponding to non existing party members will be interpreted as pokemon too!
* Things i haven't tested:
  * What happens if party size is 0.
  * What happens if party size is greater than 6.
  * What happens if it's exactly 0xFE.
 
<br><br>
 
<div align="center">

##### Pokemon Id List

</div>

* Bytes [12:19]. This is a fixed length list which contains up to 6 elements plus a terminator.
* It contains the id numbers of each pokemon in the party. The first id corresponds to the first pokemon in the party, the second id to the second pokemon, and so on..
* **Id numbers are NOT the same as pokedex numbers**. Pokemon in generation I have a hidden id number.
* A list of pokemon with their corresponding id numbers can be found here: [Bulbapedia -> List of Pokemon by index number (Generation I)](https://bulbapedia.bulbagarden.net/wiki/List_of_Pok%C3%A9mon_by_index_number_(Generation_I)).
* Note: Starting from generation II those hidden pokemon ids were replaced by the actual pokedex numbers (which makes more sense!).

<br>
  
* The list terminator (**0xFF**) goes after the id of last pokemon in the party.
* Parties with fewer than 6 pokemon will complete the list with that same terminator (0xFF).
* Values after the first list terminator are **ignored**, so it's not strictly necessary to keep sending 0xFF, any value will do **except 0xFE** (Time Capsule in Gen II fills the remaining bytes with 0x00!).
* The statement above is only valid if the [party size](#party-size) matches whatever is being sent here (e.g: a party of 3 should have 0xFF at the fourth byte).
  * The [party size](#party-size) indicates how many bytes have to be read from the list **no matter what**, so with an abnormally large party size the list terminator will be interpreted as another pokemon (and the bytes that follow it too!).
  * [Glitch] If a pokemon with 0xFF as its id is being traded then **it will become invisible** (because 0xFF actually indicates the end of the list, which seems to be used when rendering pokemon names in the trade menu). However it is still selectable! (the selection cursor is able to go below the last displayed pokemon and ends up pointing at the corresponding blank space).
    * Every pokemon below the one with 0xFF as its id will also become **invisible but selectable** (same reason, 0xFF indicates the end of the list)!
  * [Glitch] Pokemon with 0xFE as their id will completely mess up the trade data, shifting it by one byte (for each 0xFE).
* **THIS LIST CANNOT BE PATCHED**. So 0xFE shouldn't be sent (see above). That's ok because there isn't a valid pokemon whose id is 0xFE.

<br>

<div align="center">

Some examples:

<br>

A full party (Venasaur, Rhydon, Seel, Mewtwo, Mew, Charmander):

| Byte  |  0 |  1 |  2 |  3 |  4 |  5 |  6 |
|-------|----|----|----|----|----|----|----|
| Value |0x9A|0x01|0x58|0x83|0x15|0x80|0xFF|

<br>

Party with a single pokemon (Bulbasaur):

| Byte  |  0 |  1 |  2 |  3 |  4 |  5 |  6 |
|-------|----|----|----|----|----|----|----|
| Value |0x99|0xFF|0xFF|0xFF|0xFF|0xFF|0xFF|

<br>

Same party but with random numbers after the first list terminator (unusual but still valid):

| Byte  |  0 |  1 |  2 |  3 |  4 |  5 |  6 |
|-------|----|----|----|----|----|----|----|
| Value |0x99|**0xFF**|0xDE|0xCA|0xF0|0xFA|0xDE|

<br>

[Glitch] Party of 3 with 2 invisible pokemon (Bulbasaur, [invisible] **'M**, [invisible] Bulbasaur ):

| Byte  |  0 |  1 |  2 |  3 |  4 |  5 |  6 |
|-------|----|----|----|----|----|----|----|
| Value |0x99|0xFF|0x99|**0xFF**|0xFF|0xFF|0xFF|


</div>

<br><br>

<div align="center">

##### Pokemon Structure (x6)

</div>






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
