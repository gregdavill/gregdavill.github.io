---
title: "MSP430 custom ezFET Lite"
#description: <descriptive text here>
date: 2022-03-08T21:00:00+10:30
draft: false
toc: false
cover:
    image: "images/3643332-40.jpg"
    hidden: true
    relative: true
description: 
tags: []
categories: []
---

{{< bundle-image name="images/3643332-40.jpg" >}}

## Summary

Texas Instruments has a range of 16 bit microcontrollers, the MSP430. To program these require what they call a Flash Emulation Tool (FET). 


Thankfully all their affordable development kits contain a small stripped down FET, known as the ezFET Lite. TI has open sourced this design and provide both the firmware and the schematics. I was given a tag-connect cable as a giveaway through the 43oh, MSP430 community forum. I decided to create a custom PCB to connect directly to the tag-connect cable.

Here is a forum post I made when I'd designed this: [https://forum.43oh.com/topic/5530-custom-ezfet-lite/](https://forum.43oh.com/topic/5530-custom-ezfet-lite/)

## Schematic 


{{< bundle-image name="images/ezfet-sch.png" >}}

## Downloads


{{< bundle-image name="images/post-274-0-05309600-1402534643.png" >}}


This design was done in Altium when I was still studying at uni and had a "cheap" licence.
 * [ezFet_lite_tagConnect_0_01.zip](files/ezFet_lite_tagConnect_0_01.zip)

v0.1 had some issues around the inrush current and lack of bulk capacitance causing it to disconnect from the host when attached to a target.
I designed a v0.2 that had these fixes, but unfortunately did not save those files.