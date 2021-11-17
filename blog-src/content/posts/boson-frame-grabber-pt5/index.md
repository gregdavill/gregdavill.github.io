---
title: "Boson Frame Grabber Part 5"
#description: <descriptive text here>
date: 2018-11-03
draft: false
toc: false
cover:
    image: "images/IMG_5073.JPG"
    hidden: true
description: Details about the SoC architecture in the Boson Frame Grabber
tags: [boson, boson-frame-grabber, thermal]
categories: [project]
---

## RTL Architecture

The original version of the frame grabbing PCB used an iCE40 HX8K FPGA. This turned out to be a little small for the features I wanted to add. I managed to get it to a state where it would capture images from the Boson 320, but everything was hardwired together, and not easy to alter or reuse.

In order to improve module reuse and extendability a standard interface should be used. There are a few of these internal bus architectures available. I’ve decided to use wishbone, Mainly because I’ve been able to find existing wishbone modules that perform most of the major functionality of the frame grabber I’m building.

{{< bundle-image name="images/Arch_004.png" >}}

---

## Components

### [wb_intercon](https://github.com/olofk/wb_intercon)

Maybe the most important component. It is a collection of verilog components (mux, arbiter, upscaler, downscaler) and a python script that automatically generates all the required modules instantiations and wiring based on a simple easily human readable configuration file.

Documentation is a little bit lacking for this. I had to do a bit of reverse engineering on the python parser to understand the format required of the configuration file. For convenience of anyone else wanting to use this component here is a short description. There are two main configuration types, {master, slave} the master needs to know which slaves it will be connected to. The slaves need a base memory address and a length.

(Some care needs to be taken to ensure that memory segments do not overlap between slaves)

A python script takes this config file and creates two files a Verilog file and a Verilog header. This script pre-wires in muxes and arbiters as required if two masters have requested to have access over the same slave.

```
[master cpu0]
slaves =
 ram0

[slave ram0]
offset=0x00000000
size=0x4000
```

### [wb_streamer](https://github.com/olofk/wb_streamer)

The streamer translates between a stream interface and wishbone transactions. It supports configurable burst length. The stream interface is extendable to support the data from the camera directly. Which is essentially a parallel stream. Combined with some glue logic and other stream utilities (stream_upsizer, stream_dc_fifo) this makes up the main logic to capture the camera data into RAM.
### [picoSoC](https://github.com/cliffordwolf/picorv32/tree/master/picosoc)

PicoSoC is a small wrapper around picorv32, a small and robust implementation of the RISCV open source CPU architecture. The wrapper includes a SPI driver to read instruction from a SPI FLASH (typically shared with config memory). A simple UART diver, and an optional wishbone / AXI wrapper. Since the rest of my system is using the wishbone wrapper I am using the wishbone wrapper. Note this CPU does not have the greatest IPC values, so it’s more targeted as a data-path controller for there higher performance logic in your system.
### [SD controller (sdc)](https://github.com/mczerski/SD-card-controller)

This is a complete 1/4bit open source SD controller. It includes a wishbone configuration interface, and a wishbone master to read/write data blocks. The inclusion of the wishbone master is essentially a dedicated DMA.

Although it’s a complete package I did have to hack around a bit to get it working successfully. I’ll write a more detailed explanation of the changes at some point. But here are the main points.

- Buggy asynchronous FIFO implementation. Results in occasional buggy data in my testing. If sd_clk << wbclk then the original writers may not have seen this behavior. Currently I fix this by registering the next word of data. Ideally you would replace the FIFO implementation.

 - No example of how to wire into a system. It’s essential to output DATA and CMD signals at the negative edge, to ensure that setup/hold timing is satisfied on the SD cards Internal receivers. This is because data is latched on the rising edge. Registering on the neg-edge inside the I/O block is working for me.

 - No sw-driver. I’ve written my own. (work in progress) See my implementation into FatFs here. It is based of the FatFs example targeting a micro controller with real SD hardware (not SPI mode)

This full SD controller is a huge improvement of the SPI based design I was using on the old hardware. But there are still many small areas to tweak in order to boost performance.

### [HyperRAM](https://github.com/blackmesalabs/hyperram/)

My hyperRAM implementation is based off BML’s great work! I’ve used his PLL example for the Xillinx 7-series. I had to slightly modify timings for working with the DDR modules inside the Lattice ECP5. I’ve also written a basic wrapper and changed the burst read method. This supports wishbone linear bursts assuming that the master issues a request the cycle after an ACK. This is the case for the wb_streamer components, but not achievable with picosoc.

{{< bundle-image name="images/IMG_5073.JPG" >}}

I’ve hit a few issues with timing on the hyperRAM bus. For awhile it was working perfectly in hardware but the simulator was failing. If I adjusted the code to work on the simulator then the hardware would fail. I think I have tracked this down after drawing out a diagram of the signal timings.

When reading data from the HyperRAM the signals operate in “Aligned” fashion. But there is a delay of up to 5.5ns in the output of this data from the previous clock edge. I was attempting to sample this signal with Sys_clk which edges should be 5.2ns BEFORE the HyperBus clock. This also explains why my simulation was failing.

### [CMOS Capture Controller (ccc)]()

This is a module that I have written. It consists of a set of wishbone registers, and logic that operates with the Camera Clock. The main function of this module is to support gating the data stream using v-sync to support capturing a complete frame. Due to it’s position in the design I have started to add new features into this module. including clocks_per_frame, pixels_per_frame, total_frames_seen.

pixels per frame enables the hardware to automatically determine which model of camera is connected a 320 or 640 model.

---

## Current ECP5 Logic usage

With all the modules mentioned above I am just over the 12K LUT usage of the ECP5. This is about half of the total available resources. So I think I made a good choice using the ECP5-25F instead of the slightly cheaper ECP5-12F*

---
> Update:
>
> *The ECP5 is actually contains 25k LUTs inside, there is no 12k die. When making use of the Yosys/NextPnR all 25k LUTs are available
