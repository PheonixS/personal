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

![Power source connector](/assets/images/05012024-1823-power-supply.JPG){: .align-center width="250"}

Additionally, pin description on the page 110, describes how main pcb connecting to front panel, and to power source.

![Power source connector](/assets/images/05012024-1828-pin-desciption1.JPG){: width="250" }
![Power source connector](/assets/images/05012024-1828-pin-desciption2.JPG){: width="215" }

Then our connection should be like this:

- 1/FL- - from power supply to 7/VF- pin of front connector.
- 2/FL+ - from power supply to 6/VF+ pin of front connector.
- 4/VP  - from power supply to 5/Vp  pin of front connector.

Here is updated breadboard wiring:

![Breadboard](/assets/images/05012024-1851-breadboard.JPG){: .align-center width="600" }

You may notice what I added another connection: 3/CNT - 29/Raspberry PI (line is named POWER_CONTROL).

This pin used to wake up power supply from standby mode. Without +3.3V on this pin VF-/VF+/Vp(-27V) pins are not active.

## Preparation for testing of VFD display

In order to test our setup we need to understand what would be the sequence of startup required by VFD controller.

Let's check docs! VFD controller docs on [page 17](/assets/pdfs/PT6315_PrincetonTechnologyCorp.pdf) describes recommended software flowchart:

![flowchart](/assets/images/05012024-1921-flowchart.JPG){: .align-center width="300" }

**Sooo**, there are questions:

- How exactly commands are send?
- What is the command?

### Prepare power supply and front panel power

Page 6 describes commands like this:

> Commands determine the display mode and status of PT6315. A command is the first byte (b0 to b7) inputted to PT6315 via the DIN Pin after STB Pin has changed from “HIGH” to “LOW” State.

Because at this stage we don't know much about commands, default mode seems legit for us:

> When Power is turned “ON”, the 12-digit , 16-segment modes is selected.

However, from DVD 27 manual on page 86 we can see that VFD display actually has 10 digits, and only suitable configuration for VFD controller would be **10 digits, 18 segments**. Let's hold on for this knowledge for now.

We don't know however how letters are encoded, but on the page 8 there is test mode for display! Exactly what we need for testing!

![Data settings for testing](/assets/images/06012024-1911-data-format.JPG){: width="400" }

Then the command for the test mode would be like this:
`01001000` - where (starting from Most Significant Byte - MSB to Least Significant Byte - LSB):

- `01` - type of command.
- `00` - "not relevant".
- `1`  - test mode.
- `0`  - we don't care about addresses for testing, so why not put "Increment address after data has been written".
- `00`  - Write data to display mode, as we want only to display data.

Before we can use Python sketch to send SPI commands, we need to:

- Turn on front panel power supply - by setting `HIGH` on GPIO 12 (pin 32).
- Switch power source from standby mode - by setting `HIGH` on GPIO 5 (pin 29).

Go to your Raspberry PI via SSH and execute the following:

```bash
# gpio command line uses GPIO pin numbering
gpio -g mode 12 out
gpio -g mode 5 out

gpio -g write 12 1
gpio -g write 5 1
```

So now we have power supply switched from standby mode, and front panel receiving +3.3V power supply. We are ready to send some commands!

### Test mode of VFD controller

Here is a Python sketch which will send command to VFD controller:

> **Watch out!** This is incomplete code, read further to understand why.
{: .notice--warning}

```python
import spidev

bus=0
device=0

spi = spidev.SpiDev()
spi.open(bus, device)
spi.max_speed_hz = 500000
spi.mode = 0b00

#0b prefix used here to tell python to use binary notation
to_send = [0b01001011]
spi.xfer(to_send)
```

Where did I took `max_speed_hz` and `mode` parameters? That's a bit tricky:

- `max_speed_hz` - Page 19 of PT6315 - Table with electrical characteristics - Row `Oscillation Frequency` - Typical: 500 KHz or 500000 Hz
- `mode` - Page 5 - Data Input Pin: This pin inputs serial data at the rising edge of the shift clock (starting from the lower bit). This is equal to `mode 0` from [SPI Wiki page - Mode numbers table](https://en.wikipedia.org/wiki/Serial_Peripheral_Interface).

However, after executing the code above, the VFD display shows nothing. Why?

![Oscilloscope decode MSB](/assets/images/DS1Z_QuickPrint1.png){: .align-center }
Decoded SPI protocol from my oscilloscope, MSB decoder enabled
{: .text-center}

Well, the hint is above in the `mode` section:

> < . . . > This pin inputs serial data at the rising edge of the shift clock (**starting from the lower bit**)

While we sent data in MSB mode, VFD controller expects them in LSB mode!

Let's write new function which will convert MSB word to LSB.

```python
import spidev

# invert binary
def msb_to_lsb(msb_value):
    return int(bin(msb_value)[2:].zfill(8)[::-1], 2)

bus=0
device=0

spi = spidev.SpiDev()
spi.open(bus, device)
spi.max_speed_hz = 500000
spi.mode = 0b00

# 0b prefix used here to tell python to use binary notation

# command 1 - 10 digits 18 segments
spi.xfer([msb_to_lsb(0b00000110)])

# command 2 - test mode
spi.xfer([msb_to_lsb(0b01001011)])

# command 4 turn on display, first indicator will display something
# other segments might turn on as well as we did not clear controller
# ram as suggested in the documentation
spi.xfer([msb_to_lsb(0b10001111)])

# command 4 turn off segment
# spi.xfer([msb_to_lsb(0b10000111)])
```

Our sketch just displayed test data on VFD display 🎉!

![Test data](/assets/images/photo_5246811145367571896_y.jpg){: .align-center width="300" }
Segment `A-B/Track` and several other was turned on by Python sketch.
{: .text-center}

![Oscilloscope decode LSB](/assets/images/DS1Z_QuickPrint2.png){: .align-center }
Decoded SPI protocol from my oscilloscope, LSB decoder enabled
{: .text-center}

Let's continue in further posts, and actually display something meaningful on the display.

[Next article in series]({% post_url 2024-01-08-post-step3-vfd-text %})
