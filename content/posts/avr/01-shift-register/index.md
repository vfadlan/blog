---
date: 2025-12-28T05:15:34+07:00
draft: true
params:
  author: Fadlan Abduh
title: Animating LEDs with Shift Register
series: ["AVR"]
series_order: 1
---
In previous blog, we talked about how we can send a digital output signal and retrieve both digital or analog input by using the GPIO pins and managing some registers. Go ahead read it if you have not:
{{< article link="/posts/avr/00-hello/" showSummary=true compactSummary=true >}}

My ATMega328P which is the 28 pin dip version, have 23 General Purpose Input Output (GPIO) pins. For now, I consider that number as plenty, as I myself wouldn't use that much of IO.

But at some point, you'll deal with MCU that only has less than 8 GPIO pin, or maybe you have tons of sensors, outputs, etc. That moment when you have to extend your MCUs GPIO pin number, is the moment when you can make use of this thing called shift register.

## Shift Register
Yesterday we talked about how to control an output or input using register. From there, we know that a single register store the information/state about one pin. That single register can be extended to handle multiple outputs/inputs by using shift register.
There are two types of shift register, Serial in Parallel Out (SIPO) and Parallel in Serial Out (PISO). The one that we'll use in this blog is the SIPO. We can use that alongside with MCU that send serial data, and then the shift register will convert it to parallel data.

## 74HC595
