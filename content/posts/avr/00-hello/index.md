---
date: 2025-12-22T15:24:54+07:00
draft: false
params:
  author: Fadlan Abduh
title: Hello, Bare Metal AVR.
series: ["AVR"]
series_order: 0
---

![ATMega328P SMD](featured.jpg)

For the past few months, I’ve been dabbling in some low-level programming. Was focused with the C language. _The C Programming Language_ (Second Edition) by Kernighan & Ritchie is my reference and guide book. That book is the "Holy Bible" of C; it packs all the essential knowledge into less than 300 pages, whereas most technical books take more than 400. Later on, at late November, I also started working with bare-metal AVR.

This series documents my journey diving into low-level version of Arduino (ATMega328P).

## Bare Metal AVR
What is it? Bare Metal means running software directly on physical computer hardware without an excessive abstraction layer. AVR is a computer architecture for ATmega and ATTiny microcontrollers. The MCU that I use is the ATMega328P, the one used by the Arduino UNO Development Board.

I simply unplugged that chip from the Arduino Dev Board and ran my code directly on that chip. The ATMega328P can be programmed using a programmer, which is quite of a hurdle to do.

![USB ASP](/assets/img/usbasp.png "Pict. 1: USBASP Programmer, the one that I use.")

![ATMega328P Pinout Diagram](/assets/img/328ppinout.png "Pict. 2: ATMega328P 28 pin DIP Pinout. Blue-highlighted text is connected to USBASP pinouts using jumper wires.")	

## Arduino Works Perfectly Fine, Why All These Hurdles?
Well, first things first, the Arduino Development Board is a quick prototyping board that speeds up development. That’s why you’ll never see any device in the big electronic market that uses the Arduino Dev Board. It’s meant for development, not production.

Arduino run upon an Microcontroller Unit (MCU), which is the ATMega328P. There are tons of MCU families out there; STM32, PIC, MSP, Renesas, and AVR are the most commonly used MCUs in the market.

Second, it runs faster. By stripping off the Arduino abstractions, we can write a plain C program and flash it to the chip. Here’s a comparison between Arduino .ino code and its C equivalent:  
The Arduino Code:

```
void setup() {
	// setup here
}

void loop() {
	// the code that loops forever
}
```
C Code:
```
int main() {
	// setup here
	while (1) {
		// the code that loops forever
	}
}
```

You see, Arduino is just an abstraction layer. Which is good if you want high portability and fast prototyping. But the abstraction itself is slow in terms of performance.  
It won't matter if you just blink an LED every one second, but if you're doing something that requires you to have precise timing, even a millisecond matters.  

Those two are my reasons. The third is my motive: to challenge myself.

## Registers

Just like any other computer, an MCU contains a CPU (Central Processing Unit). Inside it, there’s a small high-speed memory called registers that hold data temporarily during the execution cycle of the program. These registers control how the GPIO, ADC, Timers, and any other peripherals should behave.

The structure of each register is usually specified in the corresponding MCU Datasheet or Reference Manual. For example, if we want pin 15 to blink an LED, we have to dive into the datasheet:

![ATMega328P Pinout Diagram](/assets/img/328ppinout.png "Pict 3: ATMega328P 28 pin DIP Pinout. Source: ATMega328P Datasheet.")

1. Pin 15 is identified as `PB1`. Each pin port has three registers: `DDRx`, `PORTx`, and `PINx` (x can be B, C, or D). Since we just want to output things, we can ignore the `PINx` register.
2. `DDRB`: Data Direction B Register  
	This register handles the direction of data. As you can see below, there are 8 bits of data. Each bit corresponds to a single pin (PB0-PB7). Default value is 0, which means it will act as input. Otherwise, it will act as an output.
	![DDRB Register](/assets/img/ddrbreg.png "Pict 4: DDRB Register. Source: ATMega328P Datasheet.")
		To set `PB1` as an output, we should set bit-1 at the DDRB register as `1`.
3. `PORTB` Data Register  
   The `PORTB` register handles what signal each pin should send. It can be either HIGH or LOW, 1 or 0. To turn on the LED, simply set `1` to bit-1 at the `PORTB` register.
	![PORTB Register](/assets/img/portbreg.png "Pict 5: PORTB Data Register. Source: ATMega328P Datasheet.")

## Bitwise Operators

To handle all these bit-by-bit operations I had to equip myself with the right tool: bit-wise operators. This operator simply gives me bit-wise manipulation techniques:
* Clearing a bit on a register
	```
	REG &= (~(1<<TARGET_BIT));
	```
* Setting a bit 
	```
	REG |= (1<<TARGET_BIT);
	```
* Rotating a bit
	```
	ROTATED = (REG<<3) | (REG>>(8-3))
* Extracting a bit from registers
	```
	DATA = (REG >> 4)
	```
* Many other operations

## Input
There are many ways to receive an input signal. You can receive it from some communication channel, sensor, or a simpler one, like a switch button and a potentiometer.

For handling digital input, it’s quite easy. What we do is the opposite of sending an output signal. We read the `PORTx` register.

But an analog signal is slightly more complicated than that. Firstly you have to do the conversion from analog to digital signal. In doing so, there are some registers that you should take care of: DDRx, PORTx, PINx, ADMUX, ADCSRA. And, if you manage to start the ADC process, the MCUs will store the output into two separate registers, ADCL and ADCH. They’re separated because ATMega328P have 10 bits ADC, while a register can only store up to 8 bits.

## First Project: Morse Code
I have done some easy things with this low-level stuff. But I wouldn’t call them projects; exercise is the perfect description for that.

My first project is to encode a string into visible and audible Morse code (LEDs and buzzer). The code and schematics are available on my repo, see below. You can watch it on YouTube if you want:

{{< youtubeLite id="oyK-UIJMFic" label="ATMega328P Bare Metal C Morse Code Demo" >}}

Just like any other system, here's how it works from input, process, to output:

* The input. My program will accept two digital inputs and one analog input.
	* The analog input is a signal from a potentiometer. It reads the voltage from 0v to 5v.
	* The digital inputs are simply two switches, pull-up and pull-down. It is responsible for telling the MCU when to send the Morse code. The pull-up one will make the CPU send a “Hello World” in Morse code. The other one will send the value of that analog input.
* The process. Some of the main processes are:
  1. 26 letters of alphabets -> Morse code. It only uses switch case, and call two routines: `dot()`, `dash()`. This part is the most bad-looking code in my program, but at least it works.
  2. ADC. This process will start right after the pull-down switch is triggered. The output will be stored in an int and converted to a string. Later on, it will be sent to the output.
* The output. There are two types of Morse code that will be sent by this program:
  1. “Hello World”
  2. The value of that analog input ranges from 0 (represents 0v) to 1023 (represents 5v).  

	For the ‘how’ Morse code is being sent, there are three LEDs (RGB) and a buzzer. Like I said, visible and audible. Each one indicates a different meaning:
  * Red LED blinks briefly = dot.
  * Green LED blinks shortly = dash.
  * Blue LED = separation. Separate words or indicate the starting point.

## Conclusion

The shift from high-level to low-level gives me some kind of reward that I can't explain by mere words. I'll try.

It gives me a broader yet deeper picture of how things work on a system. Stripping down the abstractions somehow makes me appreciate more about the everyday device around me. How such a simple thing has been engineered multiple times through trials and errors.

In simple terms, I am grateful for abstraction.

## Next interesting things

Thank you for spending your time reading all these yaps. It's an interesting for me to drive a device without the use of external excessive library. Especially for learning things, I'd rather write my own driver. Here are things that I might do later:
* Driving LED with Shift Register (74HC595)
* Driving 7 Segment Display with MAX7219
* Driving 8x8 Matrix LED Display with MAX7219
* Bit banging the 16x02 LCD to show text
* Bit banging in rhytm to control WS2812B addressable RGB LED

## Links
* GitHub: [vfadlan/avr-exp/04-potensio-adc](https://github.com/vfadlan/avr-exp/tree/main/04-potensio-adc)
* [ATMega328P Datasheet](https://ww1.microchip.com/downloads/en/DeviceDoc/Atmel-7810-Automotive-Microcontrollers-ATmega328P_Datasheet.pdf)
