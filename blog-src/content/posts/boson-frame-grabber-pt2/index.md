---
title: "Boson Frame Grabber Part 2"
#description: <descriptive text here>
date: 2018-06-04
draft: false
toc: false
cover:
    image: "images/IMG_2593.jpg"
    hidden: true
    relative: true
tags: [boson, boson-frame-grabber, thermal]
categories: [project]
---

In part 1, I discussed the version 1 prototype I had built. Just after ordering the PCBs for that project I decided to start thinking about miniaturizing the hardware. 
<!--more-->
{{< bundle-image name="images/IMG_2593.jpg" >}} 


{{< bundle-image name="images/Screenshot from 2021-11-17 08-33-45.jpg" alt=""  class="img-right" >}}
If I could integrate more parts into the FPGA I could reduce the BOM and PCB area required for the entire device. I was still looking at the iCE40 family and discovered the iCE40HX8K came in a 0.5mm BGA 8x8mm. This was small enough to fit directly behind the camera core!! (Notice the pocket on the back of the camera casting, it's just over 1mm deep)


## Design

{{< bundle-image name="images/bosonFrameGrabber-Front.png" alt="" class="img-left" >}} 
{{< bundle-image name="images/bosonFrameGrabber-Back.png" alt=""  class="img-right" >}}





{{< bundle-image name="images/DZhXLtkVoAAuyFE.jpg" alt="" class="img-right" >}}

The design revolves around the two main ICs, the FPGA, and the HyperRAM. These were placed on the PCB behind the camera. The connector for the camera with the same custom footprint enabled all the camera signals to be routed out on a single layer, the use of an FPGA also helped here, as most of the time you can simply swap pins to help with routing.

The FPGA is now responsible for handling the SD logic, I routed the signals required for 4bit SD mode, this can also be used in SPI mode by the FPGA. Starting with SPI mode should make this easier to bring up the firmware. 

The device features 2 user I/O. these run through a bidirectional level converter, and TVS. My thoughts here is that this will make the I/O much more robust as the level converters I'm using will easily handle input voltages of 6-7V without complaint, the same can not be said for the FPGA.

The RAM used here is the same HyperRAM as v1 prototype. 64Mbit DRAM with a 12pin HyperBus interface. I used a tiny 8Mbit QSPI Flash in 8 pin 2x3 USON package. and a JTAG interface on the back of the device which makes use of pogopins.

## FT232H programmer

{{< bundle-image name="images/Screenshot from 2021-11-17 08-35-05.jpg" alt="" class="img-right" >}}
I was going to have to develop my own programmer, fortunately Piotr had already built this great FT232H multitool! I was able to use his design and simply create my own adapter PCB!



## Hardware

{{< bundle-image name="images/IMG_2039.jpg" alt="" >}}

PCBs arrived from OSHPark. I haven't used OSHPark for PCBs since it was Laen's group PCB. (and the boards were still blue. :S ). But they are great quality, which should be expected. I ordered both the FT232H multitool design, a pogopin breakout, and my new Boson Frame Grabber PCB. 

I had ordered a PCB from OSHPark before I discovered the hyperRAM footprint error on my last PCB, so unfortunately the main PCB was useless (what are you gonna do). I was able to use the programmer, and I used the PCB for BGA practice! (I shorted a few pins on this attempt, so well worth it). Speaking of OSHPark PCBs damn do they look nice! That LPI silkscreen process really shines!

{{< bundle-image name="images/IMG_2047.jpg" alt="" class="img-left" >}}

{{< bundle-image name="images/IMG_2597.jpg" alt="" class="img-right"  >}}

## Hardware Take 2

{{< bundle-image name="images/IMG_2598.jpg" alt="" >}}

I had already ordered some new PCBs from JLCPCB.com after correcting the footprint of the HyperRAM. These boards showed up within about a week of ordering, this is of course using DHL (Otherwise your PCBs typically spend a few weeks in the mail system).

{{< bundle-image name="images/IMG_2233.jpg" alt="" >}}

The PCBs from JLCPCB look great! this was the first time I used their service. Their manufacturing specs are very impressive for the price! I'll definitely be using their service again! Although this is only a prototype run, given the size of my PCBs I decided to create a small panel using the GerberPanelizer tool (http://blog.thisisnotrocketscience.nl/projects/pcb-panelizer/)

This worked out great! JLCPCB did not have any issue routing out the space between boards. (I kept around 2mm between PCBs.)

{{< bundle-image name="images/image-asset.jpeg" alt="" >}}

{{< bundle-image name="images/IMG_2588.jpg" alt="" class="img-left" >}}

{{< bundle-image name="images/IMG_2237.jpg" alt="" class="img-left" >}}

I soldered up a single PCB by hand, this is time consuming, but forces you to relax. (If you're not relaxed while placing small parts with tweezers they tend to fling out and shoot across the room :S ). Again hot-air and flux. You can never have too much flux, I use AMTECH NC-559-V2-TF that I buy in large 30cc tubes that you can pickup from Louis Rossmann. 

{{< bundle-image name="images/IMG_2326.jpg" alt="" >}}

After assembly I really didn't expect it to work, this is the first time I've soldered a 0.5mm BGA by hand, and you can't be sure that it's correctly soldered unless you use expensive x-ray inspection tools, or destructive methods (that would destroy your prototype). I connected up my homemade programmer and powered on the board with 3.3V from my bench power supply. It worked!

A little verilog code later, tinkering with examples found online I managed to have a compiled bitstream that would flash a LED. The LED started flashing. ^_^

{{< bundle-image name="images/IMG_2353.jpg" alt="" class="img-left" >}}

{{< bundle-image name="images/IMG_2517.jpg" alt="" class="img-right" >}}

The LED blinks? Now what?

Part 3 will take a look a the verilog and firmware required to get this thing up and running. Check it out here
