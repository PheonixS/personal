---
title: "Post: VFD controller and Raspberry PI - debugging"
categories:
  - Blog
tags:
  - pcb
  - vfd
  - ic
  - raspberry
---

Let's do quick recap of what we know now:

- PT6315 is used to control the VFD display.
- SPI is used to communicate with controller.
- To power up controller, we need to supply +3V3 on pin 1 of the Front panel connector.
- We still doesn't know however how to provide power supply to VFD display itself.
- We don't know how to wake up power source.

## Breadboard testing with Raspberry PI and VFD

Let's continue our reverse engineering with tracing back required power rails for VFD display.

Schematic of power supply can be found on page 109 of [DVD 27 service manual](/assets/pdfs/dvd_27.pdf).

If we check connector on the far right part of the schematic we can find the following connector:

![Power source connector](/assets/images/05012024-1823-power-supply.JPG){: width="250" }

Additionally, pin description on the page 110, describes how main pcb connecting to front panel, and to power source.

![Power source connector](/assets/images/05012024-1828-pin-desciption1.JPG){: width="250" }
![Power source connector](/assets/images/05012024-1828-pin-desciption2.JPG){: width="215" }

Then our connection should be like this:

- 1/FL- - from power supply to 7/VF- pin of front connector.
- 2/FL+ - from power supply to 6/VF+ pin of front connector.
- 4/VP  - from power supply to 5/Vp  pin of front connector.

Here is updated breadboard wiring:

![Breadboard](/assets/images/05012024-1851-breadboard.JPG){: width="600" }

You may notice what I added another connection: 3/CNT - 29/Raspberry PI. 

This pin used to wake up power supply from standby mode. Without +3.3V on this pin VF-/VF+/Vp(-27V) pins are not active.
