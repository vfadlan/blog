---
date: 2025-12-28T05:15:34+07:00
draft: true
params:
  author: Fadlan Abduh
title: Animating LEDs with Shift Register
series: ["AVR"]
series_order: 1
---
In today's blog, we'll discuss how we can animate 8 LEDs using a shift register. I assume you already know basic things about register, we'll talk a lot about it. If you want to know what that is, we also discussed that in previous blog:
{{< article link="/posts/avr/00-hello/" showSummary=true compactSummary=true >}}

So, how to animate 8 LEDs? Simplest way is just to use 8 pins of the same port. For example, if we use Port B, we only have to handle DDRB register once and manipulating PORTB register for each frame of the animation. One problem though, the number of available pin in MCU is kind of scarce. Some MCUs even only got 12 pins including VCC and GND. 

Our case, ATMega328P actually have enough pins to handle all 8 LEDs using the same port. But let's do this the hard way for the sake of learning. That is by using a shift register.

## Shift Register
The "register" on shift register is conceptually as the same as the one we talked before. Difference is how we manipulate the ones and zeros inside those registers. And 'shift', means that the data being held by those register will be shifted to the left if the clock pin (we'll discuss this soon) is triggered.

Inside MCU, there are actually some internal shift registers. But the one that we will talk about is an external shift register in forms of Integrated Circuit. That means instead of assigning a value to a register address via C code, we manipulate those registers by sending electrical signal (HIGH or LOW) to its input pin. There are two types of shift register: Serial In Parallel Out (SIPO), and Parallel In Serial Out (PISO). We'll use SIPO since we want to minimize the use of MCU pin. 

_Note: From now on and the rest of this blog, SR stands for Shift Register. There are other meaning of SR, but that's not what I discuss in this blog._
## 74HC595 IC
### Overview
The 74HC595 is an Integrated Circuit, specifically a Serial In Parallel Out (SIPO) shift regsiter that contains 8-bits of it. Alongside the shift registers, there are also 8-bits storage register that stores binary data of 8 output pin (QA-H). The storage register will be updated to current state of shift register if the register clock pin (RCLK) is triggered.

### Pin Descriptions
![SN74HC595 DIP Pinout](img/sn74hc595-pinout.png "Pinout of SNx4HC595, taken from the TI datasheet. Note that the line above SRCLR and OE stands for Active Low, it will be active when tied to ground.")
|Pin Number|Pin Name|Description|
|:--|:--|:--|
|15,1-7|Qa-Qh|Parallel Output, fed from the storage register|
|8|GND|Ground Pin|
|9|Qh'|Serial Output for daisy chaining (we're not going to use this)|
|10|SRCLR|Shift Register Clear, active when pin tied to ground|
|11|SRCLK|Shift Register Clock, will shift the data inside SR if this pin goes from low to high (rising edge) |
|12|RCLK|Register Clock, on rising edge data inside SR will be copied to storage register, this will update the output to current state of SR|
|13|OE|Output Enable, enabled when pin connected to ground|
|14|SER|Serial data input pin|
|16|VCC|Supply voltage ranging from -0.5V to 7V|

### Functional Characteristics
![SN74HC595 Function Table](img/sn74hc595-function.png "Functional Modes of SNx4HC595, taken from the TI datasheet.")

Before we start to examine table above, there is a few explanation of those input states:
* X: Don't Care whatever the state is,
* H: HIGH,
* L: LOW,high
* ↑: Rising edge, when the pin goes from LOW to HIGH.

The ↑ is considered as a pulse clock. In fact, that's the definition of a clock, a change of state from LOW to HIGH (called the rising edge) or HIGH to LOW (falling edge). If the state goes back to previous state, it will be called Clock Pulse.

* Function 1 and 2, to enable the output pins (Qa-Qh), we shall connect OE pin to ground.  
* Function 3, we shouldn't tied SRCLR to ground since it will clear the SR.  
* Function 4, when SER is LOW and SRCLR high and we trigger SRCLK with a pulse, the shift register will be shifted from A to B, B to C and so on. The A, will be 0.  
* Function 5, the same as before but SER is HIGH. This time, the shift register value will be shifted too but the A will be 1.  
* Function 6, RCLK is triggered by rising edge, the SR data will be copied to the storage register. This will update the output (Qa-Qh).
### Nomenclature of the 7400 series IC (bonus)

## Interrupts
### A Brief Description
### Blocking vs Non-Blocking Process
### How Regular Polling Handles Blocking Process
### Interrupts in ATMega328P

## Putting it All Together
### The Animation
### Writing The Bytes to Shift Register
### Interrupt for Interface
### Just Watch.

## Sum Up
