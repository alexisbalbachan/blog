# Table of Contents
* [Motivation](#motivation)
* [Introduction](#introduction)
  * ["Normal" Remotes and the NEC Protocol](#normal-remotes-and-the-nec-protocol)
  * [The AC Paradigm Shift: Stateful vs. Stateless](#the-ac-paradigm-shift-stateful-vs-stateless)
* [Setup](#setup)
  * [Preliminary Results](#preliminary-results)
* [Identifying the AC Protocol](#identifying-the-ac-protocol)
  * [Raw Timings](#raw-timings)
  * [AI "Assistance": Confidently Incorrect](#ai-assistance-confidently-incorrect)
  * [Identification Results](#identification-results)
* [Protocol Specification](#protocol-specification)
  * [Physical Layer](#physical-layer)
  * [Payload Overview](#payload-overview)
  * [Deep Dive into the Bytes](#deep-dive-into-the-bytes)
    * [Temperature (Bits 32-35)](#temperature-bits-32-35)
    * [Operating Mode (Bits 36-37)](#operating-mode-bits-36-37)
    * [Fan Speed (Bits 16-18)](#fan-speed-bits-16-18)
    * [Timers (The First Rule Breaker)](#timers-the-first-rule-breaker)
    * [Command Type (Bits 5-7)](#command-type-bits-5-7)
  * [Macro Commands](#macro-commands)
  * [ECO Mode: The Rule-Breaking Macro](#eco-mode-the-rule-breaking-macro)
    * [Hijacking Byte 5](#hijacking-byte-5)
    * [The Synchronization Bug](#the-synchronization-bug)
  * [Pseudo-Macros](#pseudo-macros)
    * [Power OFF](#power-off)
    * [Swing Toggle](#swing-toggle)
    * [Sleep Mode: 3-Frame Payload](#sleep-mode-3-frame-payload)
  * [Follow Me](#follow-me)
    * [A Different Temperature Encoding](#a-different-temperature-encoding)
    * [Payload Isolation](#payload-isolation)
  * [DIRECT Command](#direct-command)
* [Conclusion](#conclusion)

## Motivation
It all started with a simple weekend project. I was experimenting with an old IR receiver I had salvaged from a broken device, hooking it up to see what I could intercept. To my surprise, it worked like a charm. Pointing almost any random remote I had lying around the house at it yielded instant, perfectly decoded signals. TV remotes, audio systems, etc, the software recognized them all effortlessly.

I didnt have many spare remotes just laying around so for the next experiment i tried pointing my BGH Silent Air conditioner remote to the receiver.

Instead of the expected protocol and a clean hex code command, the decoding library could only give me a wall of raw, incomprehensible numbers.

This made me question: Why was this specific remote different? Did it use a less known protocol? Maybe other libraries could decode it?

This is were i went down into the rabbit hole of AC protocols and specifically into this completely undocumented protocol.

## Introduction

### "Normal" Remotes and the NEC Protocol
Most standard remotes (TVs/VCRs) use a well documented and common protocol called **NEC**, or a close variation of it.

The concept is pretty straightforward: when you press a button, the remote sends a tiny burst of infrared light. This burst usually contains a header/signature (to wake up the receiver), an address (to identify the device, like "I am a Samsung TV"), and a command (like "Volume Up").

There are tons of variations out there from different brands, but under the hood, they are essentially the same. They mostly just change the header timings or tweak the data structure slightly.

The most important thing to note is that these commands are *very* short, usually just a few bytes of data at most. It's a quick, simple message. Honestly, that was all I wanted to know about IR protocols, and I assumed my AC remote would work exactly the same way.

To give you an idea of how simple these commands are, here is what a standard NEC "Power" command looks like (for example, an Apple TV remote):

| Header (9ms/4.5ms) | Address (8 bits) | ~Address (8 bits) | Command (8 bits) | ~Command (8 bits) | Stop Bit (Pulse) |
| :--- | :---: | :---: | :---: | :---: | :---: |
| **Start** | `0x00` | `0xFF` | `0x12` | `0xED` | 562.5µs |


<br>

And here is a variation used by Samsung TVs. As you can see, it's almost identical, instead of using 8 bit address and its compliment it now uses 16 bit addresses without compliment, also the header timings and stop bit changed.

| Header (4.5ms/4.5ms) | Address (16 bits) | Command (8 bits) | ~Command (8 bits) | Stop Bit (Pulse) |
| :--- | :---: | :---: | :---: | :---: |
| **Start** | `0x0707` | `0x02` | `0xFD` | 590µs |

<br>

How can an AC be that different? I imagined that if for example i wanted to turn the temperature up or down it would be as simple as just sending a specific command for temp up or down! Right?

Well, that was not the case...

### The AC Paradigm Shift: Stateful vs. Stateless
It turns out that Air Conditioner remotes operate on a fundamentally different philosophy than your TV remote. 

When you press "Volume Up" on a TV remote, it just sends that single, isolated command. The TV receives it, increments its internal volume counter, and that's it. The remote has no idea what the current volume is, and it doesn't care.

AC remotes, however, are **stateful**. 

When you press the "Temperature Up" button on your AC remote, it doesn't just send a "Temp Up" command. Instead, it sends the **entire state** of the remote to the AC unit. 

Every single time you press *any* button, the remote transmits a massive payload containing:
*   Power state (On/Off)
*   Operating Mode (Cool, Heat, Fan, Dry, Auto)
*   Target Temperature (e.g., 24°C)
*   Fan Speed (Low, Med, High, Auto)
*   Swing state (On/Off)
*   Timer settings

Why do they do this? Imagine you take your AC remote into another room, change the temperature from 24°C to 20°C, and then walk back into the living room and press "Fan Speed High". If the remote only sent the fan speed command, the AC would still be stuck at 24°C, while the remote's screen would say 20°C. By sending the entire state every time, the remote ensures the AC unit is always perfectly synchronized with what you see on the remote's LCD screen.

Because of this, AC protocols are significantly longer and more complex than standard NEC commands. Instead of a neat 32-bit (4-byte) package, an AC command can easily be 100 bits or more.

## Setup
When dismantling an old tv box i came up with an IR receiver module, the **VS1838B**. At the time i did not know anything else about it, but for what i could find out, it just needed to be connected to VCC/GND and it would output captured IR signals on another pin.

Much later i researched a bit more; This specific receiver is designed to detect infrared light modulated at **38 kHz**, which is the industry standard frequency used by almost all consumer electronics (including TVs, stereos, and AC units). It filters out background infrared light (like sunlight or lightbulbs) and only outputs the clean digital pulses sent by the remote. So its basically a receiver diode with supporting circuitry for filtering certain frequencies/noise and outputing digital signals.

I connected the VS1838B to an **Arduino Nano** and used the first IR library i found: [Arduino-IRremote](https://github.com/Arduino-IRremote/Arduino-IRremote) to handle the heavy lifting of reading the pulses.

Here is the core logic I used to read the incoming commands:

```cpp
#include <IRremote.hpp>

void setup() {
  Serial.begin(115200);
  IrReceiver.begin(IR_RECEIVE_PIN, ENABLE_LED_FEEDBACK);
}

void loop() {
  if (IrReceiver.decode()) {
    // 1. Print the Protocol Name
    Serial.print("Protocol: ");
    Serial.println(IrReceiver.getProtocolString());

    // 2. Print the Address (Device ID) and Command (The specific button)
    Serial.print("Address: 0x");
    Serial.println(IrReceiver.decodedIRData.address, HEX);
    Serial.print("Command: 0x");
    Serial.println(IrReceiver.decodedIRData.command, HEX);

    // 3. Check if it's unknown
    if (IrReceiver.decodedIRData.protocol == UNKNOWN) {
      Serial.println("Warning: Protocol not recognized. Using Raw Data.");
      
      // Print the raw data for debugging
      IrReceiver.printIRResultRawFormatted(&Serial, true);  
    }
    
    IrReceiver.resume(); // Enable receiving of the next value
  }
}
```

### Preliminary Results
Once everything was wired up, I grabbed the remote that originally came with the old TV box I dismantled. I pressed a few buttons, and the serial monitor immediately reacted:

```text
-----------------------
Protocol: NEC
Address: 0xFD01
Command: 0xCE
-----------------------
Protocol: NEC
Address: 0xFD01
Command: 0xD2
-----------------------
```

It worked great! The library recognized was recognizing the protocol as NEC, identified the device address (`0xFD01`), and gave me the specific hex commands for the buttons I pressed.

This worked similarly with every remote i tried, so feeling confident, i grabbed my AC remote, pointed it at the receiver, and pressed the temperature up button. Instead of a simple address and command, this monster appeared:

```text
Protocol: UNKNOWN
Address: 0x0
Command: 0x0
Warning: Protocol not recognized. Using Raw Data.
rawData[200]:
 -3276750
 +4450,-4350
 + 550,-1650 + 550,- 550 + 550,-1650 + 550,-1600
 + 550,- 550 + 550,- 550 + 550,-1650 + 550,- 500
 + 600,- 500 + 600,-1600 + 550,- 550 + 550,- 550
 + 550,-1650 + 550,-1600 + 550,- 550 + 550,-1650
 + 550,-1650 + 550,- 500 + 600,-1600 + 550,-1650
 + 550,-1600 + 600,-1650 + 500,-1650 + 550,-1600
 + 600,- 500 + 600,-1600 + 550,- 550 + 550,- 550
 + 550,- 550 + 550,- 550 + 550,- 500 + 600,- 500
 + 600,-1600 + 550,-1650 + 550,- 550 + 550,- 550
 + 550,- 500 + 600,- 500 + 600,- 500 + 600,- 500
 + 550,- 550 + 550,- 550 + 550,-1650 + 550,-1600
 + 600,-1600 + 550,-1650 + 550,-1600 + 600,-1600
 + 550,-5250 +4400,-4400 + 550,-1600 + 550,- 550
 + 550,-1650 + 550,-1600 + 600,- 500 + 600,- 500
 + 550,-1650 + 550,- 550 + 550,- 550 + 550,-1600
 + 600,- 500 + 600,- 500 + 550,-1650 + 550,-1600
 + 600,- 500 + 600,-1600 + 550,-1650 + 550,- 550
 + 550,-1600 + 600,-1600 + 550,-1650 + 550,-1600
 + 600,-1600 + 550,-1650 + 550,- 550 + 550,-1600
 + 550,- 550 + 550,- 550 + 550,- 550 + 550,- 550
 + 550,- 550 + 550,- 550 + 550,-1650 + 550,-1650
 + 500,- 550 + 550,- 550 + 550,- 550 + 500,- 600
 + 500,- 600 + 500,- 600 + 500,- 600 + 500,- 600
 + 500,-1700 + 500,-1700 + 450,-1700 + 500,-1700
 + 500,-1700 + 450,-1700 + 500
Duration=181300us
```

The library could not recognize the protocol. All it could do was dump the raw timing data and duration (in microseconds) of every single pulse (+) and space (-) it received...

## Identifying the AC Protocol

So, the data is there, but why didn't the library recognize it? Simple: It wasn't built to recognize any AC protocol. **I had to identify the protocol manually.**

My AC unit is branded as **BGH** (specifically, the "Silent Air" line). BGH is an Argentine company that primarily rebrands and distributes electronics manufactured by larger, global companies. Because of this, I was almost certain that BGH didn't invent their own proprietary IR protocol from scratch. It was highly likely they were using a pre-existing protocol from the original manufacturer of the AC unit.

I just needed to figure out *which* manufacturer actually built the electronics inside my BGH unit. It should be simple, right? **WRONG**

### Raw Timings
To make any sense of that wall of numbers, we need to understand what we are actually looking at. 

In the raw data dump, every number represents a duration in microseconds (µs). 
*   A **positive number (`+`)** represents a "Mark" or a "Pulse". This is when the IR LED on the remote is actively flashing its 38 kHz signal.
*   A **negative number (`-`)** represents a "Space". This is when the IR LED is completely off, creating a pause between the pulses.

By looking at the pattern of these pulses and spaces, we can determine the binary data (the 1s and 0s) being transmitted. In most IR protocols, a logical `0` and a logical `1` are differentiated by the length of the space that follows a standard-sized pulse. 

For example, if we look closely at the raw data, we can see a repeating pattern:
*   A short pulse (`~550µs`) followed by a short space (`~550µs`). This usually represents a **logical `0`**.
*   A short pulse (`~550µs`) followed by a long space (`~1600µs`). This usually represents a **logical `1`**.

There are also pulses that do not represent binary data. If you look at the second line of the raw data, you'll see `+4450,-4350`. This is a **Header**. In our case, it is a relatively long pulse followed by a long space. I didn't investigate this too deeply, but I suppose it acts as a signature; the receiver will only listen to the signal if it starts like this (preventing it from picking up noise or commands meant for other devices).

Then, right in the middle of the data dump, there is another "non-binary" pulse sequence: `+ 550,-5250 +4400,-4400`. 
*   The `+550,-5250` is a standard pulse followed by an exceptionally long space (around 5 milliseconds). This acts as a **Separator** or a pause.
*   Immediately after that pause, we see another `+4400,-4400`, which is a second **Header**, marking the start of a second "frame" of data.
*   I suppose that partitioning the massive payload into smaller frames makes it easier and more reliable to transmit.

In short, the raw data we just saw consists of:
1. Header 1
1. Frame 1
1. Frame Separator/End
1. Header 2
1. Frame 2

It is worth noting that these specific timings, the exact duration of the headers, the separators, and the logical 1s and 0s, are like a digital fingerprint. They vary significantly from one manufacturer to another, which is exactly what I needed to identify the true origin of my AC unit.

### AI "Assistance": Confidently Incorrect

At this point, I barely knew anything about the timings (most of what I explained in the previous section was still unknown to me). So, I took the easy route: I fed the raw timing data into an AI, asked it to convert the pulses to binary, and to identify the protocol for me.

The AI instantly and confidently identified it as the **Gree YAC** protocol. Cool! According to the AI, this protocol is **72 bits long** (more on this later) and split into two frames. It made perfect sense based on the separator I had seen.

So, I went along with it. I started changing values on the remote, bumping the temperature up, changing the fan speed, and capturing the new data. Every time I fed the new timings back to the AI, it doubled down, assuring me that I was definitely dealing with the Gree YAC protocol. 

But as I tried to map the bits, the data started making less and less sense. Changing the temperature didn't change the bits the AI said it should; instead, random bits seemed to flip all over the place. I spent days banging my head against the wall trying to force my data to fit this mold.

When I finally pushed back, the AI pivoted. It suggested that it wasn't actually Gree YAC, but rather **Gree YBOF**. Okay, fine. Let's try that. 

It made even less sense.

It turns out, the AI hadn't even bothered to count the length of my timing array. It couldn't even reliably find the differences between two captures. (In its defense, I couldn't either, analog timings have slight microsecond variations in every single capture, which makes using standard text diff tools an absolute nightmare).

Eventually, by some miracle (or perhaps just exhausting its list of guesses), the AI suggested **Midea** (or a variation of it, I can't quite remember). This protocol is **96 bits long** and sent twice. 

This time, the data actually started to align. It wasn't a perfect match, but for the first time, the bits were making a little bit of sense.

This wasn't the only time I got stuck because of AI hallucinations. I faced a lot of trouble when trying to identify temperature bits, spotting differences in binary data, and dealing with missing data in my captures. If I had just taken the long route from the start, coding a small script to make a proper diff tool and pretty-print the binary data myself, I wouldn't have wasted so much time (weeks!).

### Identification Results
Once I stopped relying on the AI and started manually comparing my raw timings against known protocol specifications, the picture became much clearer. 

The timings for the headers, the separators, and the logical 1s and 0s were indeed a near-perfect match for the **Midea** protocol family.

My conclusion is that this protocol is definitely "inspired" by Midea, if not a pure Midea itself.

Having experience in the software industry myself, I noticed that the protocol shows clear signs of unorganic growth and feature creep, often breaking its own invariants to accommodate new functionality.

## Protocol Specification
This section serves as the technical documentation for the poorly undocumented Midea/BGH Silent Air IR protocol. If you are looking to build your own transmitter, decode signals, or integrate this AC unit into a smart home system, this is the reference you need.

The protocol is fully reversed, at least for all the functions available on the original remote. There may be hidden or unused functionality, but I haven't had the time to brute-force test for it.

Most of the protocol follows a logical structure, but there are some parts where I simply could not find a pattern, making them look quite random. If you happen to figure out the logic behind these quirks, or if you have a better explanation than mine, please reach out so I can update this document!

Also, if you find that this protocol applies to your "Midea-ish" AC unit, please let me know.


### Physical Layer

As we discovered in the [Raw Timings](#raw-timings) section, this protocol uses Pulse Distance Encoding on a standard 38 kHz carrier frequency. 

To successfully transmit or decode this protocol, you must adhere to the following timing scheme:

| Signal Part | Mark (Pulse) | Space (Pause) |
| :--- | :---: | :---: |
| **Header** | 4400 µs | 4250 µs |
| **Logical `0`** | 600 µs | 450 µs |
| **Logical `1`** | 600 µs | 1550 µs |
| **Frame Separator** | 600 µs | 5100 µs |

<br>

**Bit Ordering:** Like most standard IR protocols, the data is transmitted **Least Significant Bit (LSB) first**. This means that the very first pulse/space combination you receive after the header represents Bit 0 of Byte 0.

The standard transmission consists of a **96-bit package**, which is split into two equal **48-bit frames**. 

These two frames are transmitted sequentially, separated by the 5ms Frame Separator pause. Under normal operating conditions, **Frame 2 is an exact mirror copy of Frame 1**. 

However, because of the unorganic growth we discussed earlier, there are notable exceptions to these two specific rules:
*   **The 2-Frame Rule:** The only exception is [Sleep mode](#sleep-mode-3-frame-payload) which breaks the 96-bit structure entirely by sending a third frame.
*   **The Mirror Rule:** [Sleep](#sleep-mode-3-frame-payload) also breaks the mirror rule by sending different data in Frame 2 than what was sent in Frame 1 (this is because the rule is now applied between frame 2 and 3).

We will cover these anomalies in detail in their respective sections later on.

### Payload Overview

The 48-bit frame is divided into six distinct bytes, each serving a specific purpose. The protocol employs a robust error-checking mechanism where every alternate byte acts as a checksum for the preceding one, though this rule is occasionally bent by the unorganic growth of features like [ECO mode](#eco-mode-the-rule-breaking-macro) and [Timers](#timers-the-first-rule-breaker).

I'll throw the full specification here just to illustrate how complicated it gets (don't worry, i'll explain it step by step later):

<div style="display: flex; justify-content: center;">
<div style="text-align: left;">

#### Byte 0: Header & Flags



| Bits | Purpose | Details |
| :--- | :--- | :--- |
| `0-3` | **Signature** | A constant value of `1011` (binary). |
| `4` | **Follow Me** | Flag indicating if the "Follow Me" feature is enabled (`1` = true, `0` = false). |
| `5-7` | **Command Type** | `010` for standard commands, or `101` for macro commands. |

#### Byte 1: Checksum 1
| Bits | Purpose | Details |
| :--- | :--- | :--- |
| `8-15` | **Validation** | The ones' complement of Byte 0 (`~Byte 0`). |

#### Byte 2: Fan Speed & Secondary Data
| Bits | Purpose | Details |
| :--- | :--- | :--- |
| `16-23` | **Signature** | If [Macro mode](#macro-commands), [Sleep mode](#sleep-mode-3-frame-payload), [Power OFF](#power-off), or [Swing](#swing-toggle) is enabled, this entire byte is a constant signature. |
| `16-18` | **Fan Speed** | In normal operation, dictates the [fan speed](#fan-speed-bits-16-18). |
| `19-23` | **Multiplexed Data** | Depending on the state, this can represent: <br> • A "[Follow Me](#follow-me)" temperature update. <br> • Bits `0-3` of the [Timer OFF](#timers-the-first-rule-breaker) value (Bits `19-22`), alongside a Sleep disabled flag (Bit `23` is `1` if Sleep is disabled). |

#### Byte 3: Checksum 2
| Bits | Purpose | Details |
| :--- | :--- | :--- |
| `24-31` | **Validation** | The ones' complement of Byte 2 (`~Byte 2`). |

#### Byte 4: Temperature & Mode
| Bits | Purpose | Details |
| :--- | :--- | :--- |
| `32-39` | **Signature** | If [Macro mode](#macro-commands), [Sleep mode](#sleep-mode-3-frame-payload), [Power OFF](#power-off), or [Swing](#swing-toggle) is enabled, this entire byte acts as the second part of the constant signature. |
| `32-35` | **Temperature** | The target [temperature](#temperature-bits-32-35) setting. |
| `36-37` | **Operating Mode** | The current [mode](#operating-mode-bits-36-37) (e.g., Cool, Heat, Fan). |
| `38-39` | **Multiplexed Data** | Can represent: <br> • Bits `4-5` of the [Timer OFF](#timers-the-first-rule-breaker) value. <br> • A constant `10` (Bit `38`=`1`, Bit `39`=`0`) for "[Follow Me](#follow-me)" packages. |

#### Byte 5: Checksum 3 / Timers / ECO
This byte is the most heavily overloaded, acting as a checksum in standard operation but being repurposed for advanced features.

| Bits | Purpose | Details |
| :--- | :--- | :--- |
| `40-47` | **Standard Validation** | If [Timers](#timers-the-first-rule-breaker) are disabled, it is not a "[Follow Me](#follow-me)" command, and [ECO](#eco-mode-the-rule-breaking-macro) is off: The ones' complement of Byte 4 (`~Byte 4`). |
| `40-47` | **Timers Enabled** | • Bit `40`: Copy of Bit `32`. <br> • Bits `41-46`: [Timer ON](#timers-the-first-rule-breaker) value. <br> • Bit `47`: Constant `1`. |
| `40-47` | **ECO Mode** | • Bits `40-43`: [ECO](#eco-mode-the-rule-breaking-macro) temperature. <br> • Bits `44-45`: Zero (Probably ECO mode, but in my model only COOL is allowed to use ECO). <br> • Bits `46-47`: ECO fan speed. |

#### Subsequent Frames
As mentioned earlier, the protocol typically sends two frames.

| Bytes | Frame | Details |
| :--- | :--- | :--- |
| `6-11` | **Frame 2** | Usually an exact copy of Bytes 0-5. However, if [Sleep mode](#sleep-mode-3-frame-payload) is active, this frame contains the unaltered version of what the normal settings would be. |
| `12-17` | **Frame 3** | An anomaly exclusive to [Sleep mode](#sleep-mode-3-frame-payload), containing a direct copy of Bytes 6-11. |

</div>
</div>

<br>
<hr>
<br>

Lets step back and make this a little bit easier to understand, here is what a standard, day-to-day command looks like when mapped out bit by bit. The following represents a simple state update with no macros, no "Follow Me", and no timers enabled, i think this is what the first version of this protocol looked like, before injecting features everywhere:

```text
        Byte 0
Bit   0: Signature 0 (1)
Bit   1: Signature 1 (1)
Bit   2: Signature 2 (0)
Bit   3: Signature 3 (1)
Bit   4: Follow Me (0)
Bit   5: Command Type 0 (0)
Bit   6: Command Type 1 (1)
Bit   7: Command Type 2 (0)
----------
        Byte 1
Bit   8: ~ Bit 0 (0)
Bit   9: ~ Bit 1 (0)
Bit  10: ~ Bit 2 (1)
Bit  11: ~ Bit 3 (0)
Bit  12: ~ Bit 4 (1)
Bit  13: ~ Bit 5 (1)
Bit  14: ~ Bit 6 (0)
Bit  15: ~ Bit 7 (1)
----------
        Byte 2
Bit  16: Fan Speed 0
Bit  17: Fan Speed 1
Bit  18: Fan Speed 2
Bit  19: Constant Value (1)
Bit  20: Constant Value (1)
Bit  21: Constant Value (1)
Bit  22: Constant Value (1)
Bit  23: Sleep Disabled (1)
----------
        Byte 3
Bit  24: ~ Bit 16
Bit  25: ~ Bit 17
Bit  26: ~ Bit 18
Bit  27: ~ Bit 19
Bit  28: ~ Bit 20
Bit  29: ~ Bit 21
Bit  30: ~ Bit 22
Bit  31: ~ Bit 23
----------
        Byte 4
Bit  32: Temperature 0
Bit  33: Temperature 1
Bit  34: Temperature 2
Bit  35: Temperature 3
Bit  36: Mode 0
Bit  37: Mode 1
Bit  38: Constant Value (0)
Bit  39: Constant Value (0)
----------
        Byte 5
Bit  40: ~ Bit 32
Bit  41: ~ Bit 33
Bit  42: ~ Bit 34
Bit  43: ~ Bit 35
Bit  44: ~ Bit 36
Bit  45: ~ Bit 37
Bit  46: ~ Bit 38
Bit  47: ~ Bit 39
----------
        Byte 6
Bit  48: Copy of Bit 0
Bit  49: Copy of Bit 1
Bit  50: Copy of Bit 2
Bit  51: Copy of Bit 3
Bit  52: Copy of Bit 4
Bit  53: Copy of Bit 5
Bit  54: Copy of Bit 6
Bit  55: Copy of Bit 7
----------
        Byte 7
Bit  56: Copy of Bit 8
Bit  57: Copy of Bit 9
Bit  58: Copy of Bit 10
Bit  59: Copy of Bit 11
Bit  60: Copy of Bit 12
Bit  61: Copy of Bit 13
Bit  62: Copy of Bit 14
Bit  63: Copy of Bit 15
----------
        Byte 8
Bit  64: Copy of Bit 16
Bit  65: Copy of Bit 17
Bit  66: Copy of Bit 18
Bit  67: Copy of Bit 19
Bit  68: Copy of Bit 20
Bit  69: Copy of Bit 21
Bit  70: Copy of Bit 22
Bit  71: Copy of Bit 23
----------
        Byte 9
Bit  72: Copy of Bit 24
Bit  73: Copy of Bit 25
Bit  74: Copy of Bit 26
Bit  75: Copy of Bit 27
Bit  76: Copy of Bit 28
Bit  77: Copy of Bit 29
Bit  78: Copy of Bit 30
Bit  79: Copy of Bit 31
----------
        Byte 10
Bit  80: Copy of Bit 32
Bit  81: Copy of Bit 33
Bit  82: Copy of Bit 34
Bit  83: Copy of Bit 35
Bit  84: Copy of Bit 36
Bit  85: Copy of Bit 37
Bit  86: Copy of Bit 38
Bit  87: Copy of Bit 39
----------
        Byte 11
Bit  88: Copy of Bit 40
Bit  89: Copy of Bit 41
Bit  90: Copy of Bit 42
Bit  91: Copy of Bit 43
Bit  92: Copy of Bit 44
Bit  93: Copy of Bit 45
Bit  94: Copy of Bit 46
Bit  95: Copy of Bit 47
```

### Deep Dive into the Bytes

Now that we have a high-level map of the payload, let's zoom in on the specific bits that control the core functions of the air conditioner: Temperature, Operating Mode, and Fan Speed.

<br><br>
<hr>

#### Temperature (Bits 32-35)

The target temperature is encoded in the first four bits of Byte 4 (Bits 32 through 35). The air conditioner supports a temperature range from 17°C to 30°C for all modes except Fan mode. 

If you look closely at the binary values, you'll notice they don't follow a standard sequential binary count. Instead, they resemble a non-standard Gray code.

<div align="center">

| Temperature | Bit 32 | Bit 33 | Bit 34 | Bit 35 |
| :---: | :---: | :---: | :---: | :---: |
| **17°C** | `0` | `0` | `0` | `0` |
| **18°C** | `0` | `0` | `0` | `1` |
| **19°C** | `0` | `0` | `1` | `1` |
| **20°C** | `0` | `0` | `1` | `0` |
| **21°C** | `0` | `1` | `1` | `0` |
| **22°C** | `0` | `1` | `1` | `1` |
| **23°C** | `0` | `1` | `0` | `1` |
| **24°C** | `0` | `1` | `0` | `0` |
| **25°C** | `1` | `1` | `0` | `0` |
| **26°C** | `1` | `1` | `0` | `1` |
| **27°C** | `1` | `0` | `0` | `1` |
| **28°C** | `1` | `0` | `0` | `0` |
| **29°C** | `1` | `0` | `1` | `0` |
| **30°C** | `1` | `0` | `1` | `1` |
| **NO_TEMP** | `1` | `1` | `1` | `0` |

</div>

**Why Gray Code?**
In a standard binary sequence, transitioning from `011` (3) to `100` (4) requires three bits to flip simultaneously. In a noisy IR transmission environment, if the receiver misses one of those flips, it might interpret a completely different, unintended value. Gray code ensures that only *one* bit changes between any two successive values. If you increment the temperature from 20°C (`0010`) to 21°C (`0110`), only Bit 33 flips. This provides a layer of physical signal integrity, reducing the chance of drastic temperature misinterpretations due to a single flipped bit.

**The Fan Mode Exception:**
When the AC is set to Fan mode, temperature control is disabled. In this state, the remote sends a fixed, out-of-bounds value of `1110` (Bits 32-35), which likely acts as a `NO_TEMP` flag for the unit.

<br><br>
<hr>

#### Operating Mode (Bits 36-37)

The operating mode is controlled by just two bits, located immediately after the temperature in Byte 4 (Bits 36 and 37).

<div align="center">

| Mode | Bit 36 | Bit 37 |
| :---: | :---: | :---: |
| **COOL** | `0` | `0` |
| **DRY** | `0` | `1` |
| **FAN** | `0` | `1` |
| **AUTO** | `1` | `0` |
| **HEAT** | `1` | `1` |

</div>

You might immediately spot a conflict here: **DRY** and **FAN** modes share the exact same binary value (`01`). 

How does the AC tell them apart? It relies on the [Fan Speed](#fan-speed-bits-16-18) bits to disambiguate the command. While FAN mode can be paired with any normal fan speed, DRY mode forces the fan speed to a specific, fixed value (`000`).

<br><br>
<hr>

#### Fan Speed (Bits 16-18)

The fan speed is encoded in the first three bits of Byte 2 (Bits 16 through 18). 

<div align="center">

| Speed State | Bit 16 | Bit 17 | Bit 18 | Notes |
| :--- | :---: | :---: | :---: | :--- |
| **SPEED_FIXED_VALUE** | `0` | `0` | `0` | Exclusively used to disambiguate DRY mode. |
| **SPEED 1 (Low)** | `1` | `0` | `0` | Normal operating speed. |
| **SPEED 2 (Med)** | `0` | `1` | `0` | Normal operating speed. |
| **SPEED_FOLLOW_ME** | `1` | `1` | `0` | Used only during "Follow Me" temperature updates. |
| **SPEED 3 (High)** | `0` | `0` | `1` | Normal operating speed. |
| **AUTO** | `1` | `0` | `1` | Normal operating speed. |
| **SPEED_OFF** | `0` | `1` | `1` | Used only when powering off the unit. |
| **SPEED_MACRO** | `1` | `1` | `1` | Used only during Macro commands. |

</div>

**The Signature Theory:**
While it's tempting to view `SPEED_OFF` and `SPEED_MACRO` as dedicated fan speeds, I prefer to interpret them as artifacts of the protocol's unorganic growth. 

When you [power off](#power-off) the unit or send a [macro](#macro-commands), the protocol doesn't just change the fan speed; it overwrites the entirety of Byte 2 (and Byte 4) with a fixed signature. Because these bits happen to fall within the Fan Speed index, they look like special speed values. However, since other unrelated bits (like the temperature) are also overwritten with nonsensical values during these commands (e.g., the temperature might read as 28°C during a power-off command), it makes more sense to treat the entire byte as a hardcoded signature rather than a collection of individual state flags.

<br><br>
<hr>

#### Timers (The First Rule Breaker)

Timers are where the protocol starts to show its seams. Up until this point, the protocol followed a strict rule: every odd byte (Byte 1, 3, 5) is the exact ones' complement of the preceding even byte (Byte 0, 2, 4). This acts as a robust checksum mechanism. 

Timers break this rule entirely. 

Why? Most likely because the engineers designing the protocol simply ran out of space in the standard payload and had to cram the timer data wherever it would fit.

The AC supports two types of timers: **Timer ON** (turn on after X hours) and **Timer OFF** (turn off after X hours). The values can be set in 0.5-hour increments up to 10 hours, and then in 1-hour increments up to 24 hours.

**Data Placement:**
*   **Timer ON:** Stored contiguously in 6 bits within Byte 5 (Bits 41 to 46).
*   **Timer OFF:** Stored non-contiguously across two different bytes. The first 4 bits are in Byte 2 (Bits 19 to 22), and the last 2 bits are in Byte 4 (Bits 38 and 39).

**Breaking the Checksum:**
If *either* timer is active, the protocol intentionally breaks the ones' complement rule for Byte 5. Specifically, Bit 40 is forced to be an exact copy of Bit 32 (which is part of the temperature data), rather than its complement. 

<br>
<div align="center">
  <strong>⚠️ IMPORTANT: If <em>any</em> timer is enabled, <code>Bit 40 == Bit 32</code>, and Byte 5 completely breaks the complement rule. ⚠️</strong>
</div>
<br>

When both timers are disabled, Byte 5 returns to its normal behavior (acting as the complement of Byte 4). In this disabled state, the Timer OFF bits (19-22) default to `1111`, and bits 38-39 default to `00`. 

If only one timer is enabled, the disabled timer is set to a special "off" value of `111111` (all six bits set to 1).

Both timers cannot be set to the same value, i'm not sure if this is a protocol restriction or if my model just doesn't allow it. If this happens one of the values is pushed to the next (or previous) value on the list before sending the command.

**Timer Value Mapping:**
Here is the binary mapping for the timer values. Note how the bits are mapped depending on whether it is a Timer ON or Timer OFF command:

</br>
<div align="center">


*(Note: I placed bits 41/42 and 38/39 as the Most Significant Bits in the table below, but my ordering is arbitrary. It might make more sense to place them as the Least Significant Bits, but this is how I originally mapped them out.)*

| Hours | Timer ON Bits <br> `43` `44` `45` `46` `41` `42` <hr> Timer OFF Bits <br> `19` `20` `21` `22` `38` `39` |
| :---: | :---: |
| **0.5** | `0 0 0 0 0 0` |
| **1.0** | `0 0 0 1 0 0` |
| **1.5** | `0 0 1 0 0 0` |
| **2.0** | `0 0 1 1 0 0` |
| **2.5** | `0 1 0 0 0 0` |
| **3.0** | `0 1 0 1 0 0` |
| **3.5** | `0 1 1 0 0 0` |
| **4.0** | `0 1 1 1 0 0` |
| **4.5** | `1 0 0 0 0 0` |
| **5.0** | `1 0 0 1 0 0` |
| **5.5** | `1 0 1 0 0 0` |
| **6.0** | `1 0 1 1 0 0` |
| **6.5** | `1 1 0 0 0 0` |
| **7.0** | `1 1 0 1 0 0` |
| **7.5** | `1 1 1 0 0 0` |
| **8.0** | `1 1 1 1 0 0` |
| **8.5** | `0 0 0 0 0 1` |
| **9.0** | `0 0 0 1 0 1` |
| **9.5** | `0 0 1 0 0 1` |
| **10.0** | `0 0 1 1 0 1` |
| **11.0** | `0 1 0 1 0 1` |
| **12.0** | `0 1 1 1 0 1` |
| **13.0** | `1 0 0 1 0 1` |
| **14.0** | `1 0 1 1 0 1` |
| **15.0** | `1 1 0 1 0 1` |
| **16.0** | `1 1 1 1 0 1` |
| **17.0** | `0 0 0 1 1 0` |
| **18.0** | `0 0 1 1 1 0` |
| **19.0** | `0 1 0 1 1 0` |
| **20.0** | `0 1 1 1 1 0` |
| **21.0** | `1 0 0 1 1 0` |
| **22.0** | `1 0 1 1 1 0` |
| **23.0** | `1 1 0 1 1 0` |
| **24.0** | `1 1 1 1 1 0` |
| **OFF** | `1 1 1 1 1 1` |

</div>





<br><br>
<hr>

#### Command Type (Bits 5-7)

The protocol distinguishes between two primary types of commands, dictated by the values in Bits 5 through 7 of Byte 0.

*   **Standard Commands (`010`):** These are your day-to-day operations. [Temperature](#temperature-bits-32-35) adjustments, [mode](#operating-mode-bits-36-37) changes, [fan speed](#fan-speed-bits-16-18) selections, and [timer](#timers-the-first-rule-breaker) configurations all fall under this category. They coexist within the standard payload structure we've been analyzing, where each bit has a dedicated, persistent meaning.
*   **Macro Commands (`101`):** These are special, usually toggle-based functions. Features like **TURBO**, **ECO mode**, **FRESH** (ionizer), **SELF CLEAN**, and **LED** (display toggle) are transmitted as macros. 

When the Command Type bits are set to `101`, the entire meaning of the payload shifts. The standard structure is temporarily abandoned. For instance, Bits 32-35 no longer represent the target temperature; instead, they become part of a specific macro signature. The data structure for these macros is entirely different and will be detailed in its own dedicated section.

**Why the Distinction?**
There isn't a strict technical necessity for this separation. Any of these macro functions could theoretically have been integrated into the standard payload as a single toggle bit. The most likely explanation is, once again, unorganic growth. The developers probably introduced these macro features in a later hardware revision, long after all the bits in the standard payload had already been assigned and locked in. Instead of redesigning the entire protocol, they simply created a new "Command Type" branch to handle the overflow.


<br><br>
<hr>

### Macro Commands

As mentioned in the [Command Type](#command-type-bits-5-7) section, when Bits 5-7 are set to `101`, the protocol enters **Macro Mode**. In this state, the standard payload structure is discarded, and the entire transmission is dedicated to executing a **single** macro action. 

During a macro command, Byte 0 still holds the signature and command type, and Bytes 1, 3, and 5 generally continue to act as ones' complements (though [ECO mode](#eco-mode-the-rule-breaking-macro) breaks this rule by utilizing Byte 5). The actual payload of the macro is carried entirely within **Byte 2** and **Byte 4**.

I couldn't find a logical bit-level meaning for the data within these macros. The bits seem almost entirely random. Because of this, it's much easier to treat them as hardcoded **signatures** rather than individual state flags.

Here is how it works: For every macro command, **Byte 2 is always a constant `10101111`** (`0xAF`). The specific action to be performed is then determined entirely by the signature found in **Byte 4**:

<div align="center">

| Macro Action | Byte 4 Signature (Binary) | Byte 4 (Hex) |
| :--- | :---: | :---: |
| **ECO ON** | `01000001` | `0x41` |
| **ECO OFF** | `11000001` | `0xC1` |
| **TURBO** | `01000101` | `0x45` |
| **SELF CLEAN** | `01010101` | `0x55` |
| **FRESH (Ionizer)** | `11000101` | `0xC5` |
| **LED DISPLAY** | `10100101` | `0xA5` |

</div>

With the exception of ECO ON/OFF, where the Most Significant Bit (MSB) clearly represents the state change (`0` for ON, `1` for OFF), the rest of the bits appear arbitrary. There might be an underlying logic to their arrangement, but if there is, I haven't cracked it yet.

Interestingly, this structure leaves enough room for hundreds of potential macros. It's highly likely there are hidden or debug commands not utilized by my specific remote model. Did the engineers finally decide to future-proof the protocol? Perhaps!

To summarize: If the Command Type is `101` and Byte 2 is exactly `10101111`, the AC unit will look at Byte 4 to determine which macro to execute.

**A Note on ECO Mode:** [ECO mode](#eco-mode-the-rule-breaking-macro) is a special case. It will get its own dedicated section later on because it hijacks Byte 5 to store additional settings, breaking the standard checksum rule.

**A Note on Power Off, Sleep, and Swing:** You might be wondering why actions like [Power OFF](#power-off), [Sleep](#sleep-mode-3-frame-payload), or [Swing](#swing-toggle) aren't on this list. I do not classify them as macros because their Command Type is still set to `010` (Standard Command). They are essentially "normal" commands that just happen to behave like macros by overwriting large chunks of the payload. My theory is that these were the "OG" macros, implemented long before the dedicated `101` Command Type revision was introduced. They will also be covered in their own respective sections.


<br><br>
<hr>

### ECO Mode: The Rule-Breaking Macro

ECO Mode is a fascinating anomaly. It is technically a [Macro Command](#macro-commands), but it is the *only* macro that hijacks Byte 5 to store additional state information, completely breaking the standard checksum rule for that byte.

As with all macros, Byte 2 contains the hardcoded signature `10101111` (`0xAF`). The specific ECO action is then defined in Byte 4. You can think of this as two distinct signatures, or a single signature where the Most Significant Bit (MSB) acts as a toggle:

<div align="center">

| ECO Action | Byte 4 Signature (Binary) | Byte 4 (Hex) |
| :--- | :---: | :---: |
| **ECO ON** | `01000001` | `0x41` |
| **ECO OFF** | `11000001` | `0xC1` |

</div>

On my specific AC model, ECO mode can only be activated while the unit is in COOL [mode](#operating-mode-bits-36-37). Furthermore, if the current target [temperature](#temperature-bits-32-35) is set below 24°C when ECO is activated, the remote will automatically force the temperature up to 24°C.

#### Hijacking Byte 5

The most interesting aspect of the ECO command is what happens in Byte 5. Instead of acting as a checksum for Byte 4, it is repurposed to store a snapshot of the AC's current state:

<div align="center">

| Bits | Purpose | Details |
| :--- | :--- | :--- |
| `40-43` | **Temperature** | The target temperature (24°C to 30°C), encoded exactly like a [standard temperature](#temperature-bits-32-35). |
| `44-45` | **Mode (Unconfirmed)** | Always `00` in my captures. Since `00` corresponds to COOL mode, and my remote only allows ECO in COOL mode, it's highly probable these bits represent the operating mode. |
| `46-47` | **Fan Speed** | The current fan speed, encoded exactly like a [standard fan speed](#fan-speed-bits-16-18). I have only observed normal speeds (Low, Med, High, Auto) here. |

</div>

#### The Synchronization Bug

Why bother storing all these settings inside a one-off macro command? At first glance, it seems entirely redundant. The AC unit already knows its current temperature and fan speed; it doesn't need the remote to remind it during an ECO toggle.

The answer lies in how the AC handles state changes *while* ECO mode is active. If you change the [fan speed](#fan-speed-bits-16-18) (or [temperature](#temperature-bits-32-35)?) while ECO is running, the AC unit will automatically deactivate ECO mode. 

Theoretically, the remote should handle this gracefully: if you change the fan speed, the remote should transmit an `ECO OFF` macro, embedding the *new* fan speed inside Byte 5. This would simultaneously disable ECO and update the fan speed in a single, clean command.

However, **the remote itself does not follow its own protocol, leading to a desynchronization bug!**

When you change the fan speed while ECO is active, the remote doesn't send an `ECO OFF` macro. Instead, it just sends a standard state update with the new fan speed. The AC unit receives this standard update and correctly decides to deactivate ECO mode. 

The problem? **The remote's internal state still thinks ECO is active**. 

This leads to a frustrating user experience:
1. You press **ECO** (AC activates ECO, Remote knows ECO is active).
2. You change the **Fan Speed** (AC deactivates ECO, Remote *still* thinks ECO is active).
3. You press **ECO** again, expecting it to turn back on. Instead, the remote sends an `ECO OFF` command (because it thought it was still on). The AC does nothing, but the remote finally clears the ECO state from its memory.
4. You have to press **ECO** a *third* time to actually turn it back on.

It's a classic case of state desynchronization between a stateless receiver and a stateful remote, caused by a firmware oversight in the remote itself.


<br><br>
<hr>

### Pseudo-Macros

There are certain commands that behave like macros by overwriting large chunks of the payload with hardcoded signatures, but their [Command Type](#command-type-bits-5-7) remains set to `010` (Standard Command). Because they don't share the universal `10101111` Byte 2 signature used by true macros, they are classified here as "Pseudo-Macros". This category includes Power OFF, Swing, and Sleep mode.

<br><br>
<hr>

#### Power OFF

The Power OFF command is an interesting hybrid. It behaves exactly like a [Macro Command](#macro-commands), overwriting the standard payload with hardcoded signatures, but its [Command Type](#command-type-bits-5-7) is set to `010` (Standard Command) rather than `101` (Macro Command). 

Because it doesn't share the universal `10101111` Byte 2 signature used by true macros, I classify it in its own distinct category. Instead, it uses a unique signature pair across both Byte 2 and Byte 4:

<div align="center">

| Action | Byte 2 Signature | Byte 4 Signature |
| :--- | :---: | :---: |
| **POWER OFF** | `11011110` (`0xDE`) | `00000111` (`0x07`) |

</div>

You may have seen the `SPEED_OFF` speed in the [Fan Speed section](#fan-speed-bits-16-18), well, the power off signature just happens to overwrite the fan speed bits in a way that an unique fan speed gets placed in those bits. So there's some kind of logic behind these signatures.

It is important to note that this is strictly a **Power OFF** command, not a power toggle. If you send this command while the AC unit is already off, the unit will simply beep to acknowledge receipt, but it will remain off. 

To turn the unit *on*, the remote doesn't send a "Power ON" macro. Instead, it simply transmits a standard state update containing the current [temperature](#temperature-bits-32-35), [mode](#operating-mode-bits-36-37), and [fan speed](#fan-speed-bits-16-18). 

*(Note: Sending true macro commands to a powered-off unit appears to have no effect, though I haven't fully tested how the unit responds to [Timer ON/OFF](#timers-the-first-rule-breaker) updates while in a powered-down state.)*

<br><br>

#### Swing Toggle

The Swing command is perhaps the least complex of the special functions. Much like Power OFF, it behaves like a [Macro Command](#macro-commands) by overriding the payload with signatures, but it retains the `010` (Standard Command) [Command Type](#command-type-bits-5-7).

It also possesses its own unique signature pair, completely distinct from the standard macro family:

<div align="center">

| Action | Byte 2 Signature | Byte 4 Signature |
| :--- | :---: | :---: |
| **SWING TOGGLE** | `11010110` (`0xD6`) | `00000111` (`0x07`) |

</div>


To turn Swing ON or OFF you send exactly the same command, the AC unit will toggle the current state when it receives it.

The only truly noteworthy characteristic of the Swing command is its interaction with Sleep mode. As we will explore in the upcoming [Sleep mode](#sleep-mode-3-frame-payload) section, Swing is the *only* special action that can be transmitted and executed without overriding or canceling an active Sleep state.



<br><br>

#### Sleep Mode: 3-Frame Payload

Sleep mode was, without a doubt, the most difficult feature to reverse-engineer. The primary issue wasn't the logic, but the hardware. 

My initial setup using an Arduino Nano and the standard IR library was hard-capped at capturing 200 timings (roughly 100 bits). Furthermore, the VS1838B receiver has an automatic gain control (AGC) circuit that adjusts its sensitivity based on ambient light. Because the Sleep command is so incredibly long, the receiver would interpret the prolonged burst of IR light as background noise, lower its sensitivity mid-transmission, and drop the final pulses. I had to upgrade to an ESP32 with a larger buffer and physically dim the IR LED on the remote to get a clean capture.

<br>
<div align="center">
  <strong>⚠️ WARNING: While Sleep mode is active, the remote transmits 3 frames of data instead of 2, totaling 144 bits. This officially makes the protocol variable-length. ⚠️</strong>
</div>
<br>

Like [Power OFF](#power-off) and [Swing](#swing-toggle), Sleep behaves like a [Macro Command](#macro-commands) but retains the `010` (Standard Command) [Command Type](#command-type-bits-5-7). It uses its own unique signature pair:

<div align="center">

| Action | Byte 2 Signature | Byte 4 Signature |
| :--- | :---: | :---: |
| **SLEEP MODE** | `00000111` (`0x07`) | `11000000` (`0xC0`) |

</div>

On my specific model, Sleep can only be activated in COOL, HEAT, or AUTO [modes](#operating-mode-bits-36-37), and it forces the [fan speed](#fan-speed-bits-16-18) to AUTO. Changing the fan speed or operating mode will immediately deactivate Sleep. However, looking at the protocol structure, there is no technical reason for these restrictions, they are likely just arbitrary rules enforced by the remote's firmware.

##### Breaking the Mirror Rule

The most significant change during Sleep mode is that **Frame 2 is no longer a mirror copy of Frame 1**. 

Instead, Frame 1 is entirely hijacked by the Sleep signatures. Frame 2 then carries the *actual* standard payload (the current temperature, mode, timers, etc.). Finally, a newly appended **Frame 3** acts as the mirror copy of Frame 2.

You can conceptualize this in two ways:
1.  Frame 1 is overwritten by Sleep, and a 3rd frame is added to satisfy the mirror requirement for the actual data in Frame 2.
2.  A new "Frame 0" containing the Sleep signature is prepended to the standard 2-frame transmission.

Here is what a complete Sleep payload looks like:

<div align="center">

| Frame | Byte | Contents |
| :---: | :---: | :--- |
| **Frame 1** | `0` | Standard Header + Command Type (`010`) |
| | `1` | `~Byte 0` |
| | `2` | **Sleep Signature** (`0x07`) |
| | `3` | `~Byte 2` |
| | `4` | **Sleep Signature** (`0xC0`) |
| | `5` | `~Byte 4` |
| **Frame 2** | `6` | Standard Header + Command Type (`010`) |
| | `7` | `~Byte 6` |
| | `8` | Actual Fan Speed + Timer OFF bits |
| | `9` | `~Byte 8` |
| | `10` | Actual Temperature + Mode |
| | `11` | `~Byte 10` (or Timer ON bits) |
| **Frame 3** | `12-17` | Exact copy of Frame 2 |

</div>

Turning off Sleep mode is as simple as sending a standard 2-frame status update.

##### The Swing + Sleep + Timer Edge Case

Remember when I mentioned that [Swing](#swing-toggle) is the only special action that can be combined with Sleep? Here is how that interaction works:

When both are active, Sleep hijacks Frame 1, and Swing hijacks Frame 2. The actual state data is pushed all the way back to Frame 3.

This creates a fascinating vulnerability in the protocol's error-checking. If you have [Timers](#timers-the-first-rule-breaker) enabled (which breaks the Byte 5 checksum) *and* you send a [Swing](#swing-toggle) command while in Sleep mode (which breaks the frame mirroring), the critical temperature and mode data in Byte 4 is left with **zero checksum redundancy**. If a bit flips in transit, the AC unit has no way to verify the data and might set itself to an unintended temperature!

<br><br>
<hr>

### Follow Me

The "Follow Me" feature is yet another inconsistency in the protocol. It is neither a true [Macro Command](#macro-commands) nor a standard state update. While its [Command Type](#command-type-bits-5-7) remains set to `010` (Standard Command), it doesn't hijack the entire frame with a signature like a macro would. Instead, it only overwrites specific parts of the payload.

This mode is used by the remote to periodically transmit its own local ambient temperature reading to the AC unit, allowing the unit to adjust its cooling or heating based on where the remote is located in the room.

<br>
<div align="center">
  <strong>⚠️ WARNING: Whenever a Follow Me command is transmitted, Bit 4 of Byte 0 is explicitly set to <code>1</code>. ⚠️</strong>
</div>
<br>

There are two distinct types of Follow Me payloads: an initial activation command, and the subsequent automatic periodic updates sent by the remote every few minutes. 

The protocol distinguishes between these two message types by hijacking the [Fan Speed](#fan-speed-bits-16-18) bits:
1.  **Activation Command:** If the fan speed bits are set to `SPEED 2` (`010`), the command indicates that the Follow Me feature has just been turned on by the user.
2.  **Automatic Update:** If the fan speed bits are set to `SPEED_FOLLOW_ME` (`011`), the command is an automatic ambient temperature report.

#### A Different Temperature Encoding

In both types of Follow Me messages, the remote's local temperature reading is embedded in **Bits 19 through 23** of Byte 2. This hijacks the space normally reserved for the [Timer OFF](#timers-the-first-rule-breaker) bits, plus Bit 23 (which is otherwise rarely used, except as part of the [Sleep signature](#sleep-mode-3-frame-payload), and defaults to `1`).

What makes this truly bizarre is the encoding. Unlike the standard target temperature (which uses a [Gray code-like structure](#temperature-bits-32-35)), the Follow Me temperature is transmitted in **plain binary**. Why the engineers chose to use two completely different temperature encodings within the same protocol is a mystery. 

Furthermore, the bit ordering appears to be reversed compared to the rest of the protocol, with the Least Significant Bit (LSB) at Bit 23 and the Most Significant Bit (MSB) at Bit 19.

Finally bit 38 is set to 1 and bit 39 to 0, i don't know why or if they're really necessary, originally they're part of the timer OFF bits so maybe they're set that way prevent any confusion with authentic timer off values. (in a normal payload, when both timers are disabled, bits 38 and 39 are set to 00. When they're enabled byte 5 breaks the compliment rule, so there's no way to get the FOLLOW ME message mixed up).

#### Payload Isolation

Follow Me messages appear to be completely isolated from the rest of the AC's state. I haven't tested if the AC unit would even read the standard target temperature bits during a Follow Me update, but it's highly likely they are ignored.

For example, if [Sleep Mode](#sleep-mode-3-frame-payload) is active alongside Follow Me, manually changing the temperature on the remote will correctly transmit the massive 3-frame Sleep payload. However, when the remote automatically wakes up to send a Follow Me temperature update, it only transmits a standard 2-frame payload, completely ignoring the active Sleep state.

In short: Follow Me is a highly specialized, isolated message flagged by Bit 4. It lacks a universal signature, hijacks the fan speed to indicate its message type, and uses a completely unique binary encoding for temperature in Byte 2.



<br><br>
<hr>

### DIRECT Command

The "DIRECT" function is arguably the most bizarre anomaly in the entire protocol. It completely breaks every rule established so far. Instead of sending the standard payload (whether a normal state update or a [Macro Command](#macro-commands)), it abandons the BGH/Midea protocol entirely and transmits the command using a completely different protocol: **SAMSUNG48**.

Samsung48 is a variation of the standard NEC protocol, which transmits a 16-bit address followed by a 16-bit command (along with its logical complement to act as a checksum).

When you press the DIRECT button, the remote transmits the following Samsung48 message:
*   **Address:** `0xB24D`
*   **Command:** `0x07F0`

I attempted to brute-force other Samsung48 commands to see if the AC unit would recognize them, but the process is incredibly time-consuming due to the unit's slow response time to rapid commands.

**Why use a different protocol?**
My theory is that this shorter, simpler protocol was chosen specifically for the DIRECT function because it is the *only* button on the remote that you are expected to press multiple times in rapid succession (to manually step the louver to a specific angle). 

Using a lightweight 48-bit protocol instead of the massive 96-bit (or 144-bit) standard payload has two major benefits:
1.  **Responsiveness:** It allows the remote to transmit the command immediately upon every button press. If you rapidly press the temperature buttons, the remote actually waits for you to stop pressing for about a second before it finally transmits the final state. The DIRECT button needs to be instantaneous.
2.  **Battery Life:** Transmitting a short burst of IR data repeatedly consumes significantly less power than transmitting the full, complex payload over and over again.

<br><br>
<hr>

## Conclusion

Decoding the Midea/BGH Silent Air IR protocol was a massive undertaking that took weeks of trial, error, and staring at raw microsecond timings. What started as a simple test to read my AC remote signals quickly spiraled into a deep dive into the chaotic world of embedded firmware.

This remote is a fascinating case study in how engineering requirements evolve (and sometimes devolve) over time. It is clear that this protocol was not designed all at once, but rather patched and expanded over multiple hardware revisions, leading to the quirks and inconsistencies we see today.

I hope this documentation serves as a valuable resource for anyone looking to build their own transmitters, decode signals, or simply understand the hidden complexity behind a simple button press. If you have any questions, notice any errors, or discover that this protocol applies to your non-BGH AC unit, please reach out! I would love to hear about your findings and continue updating this reference.

