---
title: "Post: VFD controller and wiring"
categories:
  - Blog
tags:
  - pcb
  - vfd
  - ic
---

My journey started with analyzing original wiring of [DVD 27](/assets/pdfs/dvd_27.pdf).

I was particularly interested in:

1. What VFD controller is on board?
2. Which protocol is used to communicate with it?
3. How to power up controller?
4. How was the VFD receiving negative power supply?

## Reverse engineering

Page 107 of service manual turns out to be a good source of information (my sincere gratitude towards Harman/Kardon engineers designing this - schematic is clear and well structured).

Let's go one by one over questions:
According to the schematic [PT6315 VFD driver](/assets/pdfs/PT6315_PrincetonTechnologyCorp.pdf) is used to control VFD.

![Type of controller in use](/assets/images/04012024-vfd-controller-ssbbaa.JPG){: width="400" }

Next question was how to communicate with controller. You probably can already tell based on the pins: 9/STB, 8/CLK, and 7/DIN. But I didn't know that at the time.

Page 1 of PT6315 datasheet describes capabilities, which are (some omitted):

> • Serial Interface for Clock, Data Input, Data Output, Strobe Pins

I have to google it tbh - because, by some reason, "Serial Interface" in my head was related to RS232 interface.

Anyway, at some point I found that Serial Interface actually mean [Serial Peripheral Interface](https://en.wikipedia.org/wiki/Serial_Peripheral_Interface).
