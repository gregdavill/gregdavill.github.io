---
title: "Allwinner S3 Video Encoders"
#description: <descriptive text here>
date: 2022-07-12T21:00:00+10:30
draft: false
toc: false
cover:
    image: "images/002.jpeg"
    hidden: true
    relative: true
description: 
tags: []
categories: []
---

{{< bundle-image name="images/002.jpeg" >}}

## Introduction

I've been using an Allwinner S3 SoC for experiments with the FLIR Boson. The description from the website sums the device up nicely:

> S3 is a high cost-effective video encoding processor jointly developed by Zhuhai Allwinner Technology Co., Ltd. ("Allwinner") and Sochip Technology Co., Ltd. ("Sochip").
> Its built-in single-core ARM Cortex-A7 runs at 1.2GHz with 1Gbit DRAM (DDR3) , supporting numerous peripheral devices. The Chip package is 234 balls FBGA package, size 11mm x 11mm , suitable for various types of product development. 
> --- http://www.sochip.com.cn/s3/index.php?title=What_is_S3_%3F


{{< bundle-image name="images/Cpuarch.png" >}}

What was appealing to me for use with the Boson thermal camera is the compact BGA package, the internal DRAM, and embedded ethernet 100 Mbit PHY.
Interestingly the CPU used inside is the same as the V3/V3s/S3, there is mainline kernel and uboot support. This is good because I'm very much an embedded Linux beginner.

After following the reference design provided by SOChip, along with some other great OSHW projects:
* [https://github.com/Jana-Marie/OtterCam-s3](OtterCam-s3)
* [https://github.com/Ottercast/OtterCastAudioV2](OtterCast) 
* [https://github.com/OLIMEX/S3-OLinuXino](S3-OLinuXino)

I had a custom board designed. After some quick assembly I had a board that powered up.
Soon after that with a custom buildroot project, and had linux 5.18 booted on the board. 🙌

{{< bundle-image name="images/001.jpeg" >}}

## H264 Encoder

After some initial testing with libopenh264 with ffmpeg the performance is not great. Achieving only a few fps encoding an incoming USB UVC stream into h264. Not a dealbreaker for this project.

The S3 contains a hardware encoding block that claims performance of 120FPS, this block is the same across many Allwinner parts. One such part is the H3. Unfortunately it's not documented by Allwinner.
If you want to make use of the codec you'll need to use the older kernel provided by Allwinner, and their cedar core.

Cedrus is an open source alternative that makes use of community documentation of the codec registers. It's available upstream in the kernel. Great! Except it only handles decoding not encoding. :(

---

Searching around I found some older posts talking about hw encode on the Allwinner H3.

{{< bundle-image name="images/004.png" >}}
> https://forum.armbian.com/topic/11551-4kp30-video-on-orange-pi-lite-and-mainline-hardware-acceleration/page/2/#comment-89815

They mentioned these components: 
* https://github.com/uboborov/sunxi-cedar-mainline 
* https://github.com/uboborov/ffmpeg_h264_H3
* https://github.com/uboborov/cedrus

### sunxi-cedar-mainline 
This is a kernel module which when loaded creates `/dev/cedar_ve`. This file facilitates the interaction from a userspace app to the low level codec block.
What is interesting is that this project specifically list Kernel 5.7, with config. It makes use of the CMA, Continuous Memory Allocator, a part of the Linux mainline Kernel. Many repos for "cedar" seem to use an older ION system from Android, and are pinned to old kernel versions. 

### ffmpeg_h264_H3
This is a custom encoder that compiles into FFmpeg. Unfortunately this version does make use of ION, which creates a bit of a disconnect between the kernel module and the userspace code... :/

### cedrus
These small utilities are super handy, they're small single purpose applications `h264enc`, `jpeg-test`, `mpeg-test`. 

The code makes use of the CMA, and also the functions and code layout match up pretty perfectly to the FFmpeg encoder code. Combining this code into the FFmpeg looks like it might be the way to go.

## Building

So it looks like we have all the pieces for this puzzle but can we put them together..

A very simple [package](https://github.com/gregdavill/boson-s3-dev-sw/tree/7368333f0b5bcfb46ecb01c36bdba9204f738d78/buildroot/package/cedar) is added buildroot project to compile the kernel module, this also installs the module into the target filesystem.
```txt
################################################################################
#
# cedar-mainline
#
################################################################################

CEDAR_VERSION = cb9ea3d9894fd88a5b7f67fea220ceaa5ed17656
CEDAR_SITE = $(call github,gregdavill,sunxi-cedar-mainline,$(CEDAR_VERSION))
CEDAR_LICENSE = GPL-2.0
CEDAR_LICENSE_FILES = LICENSE

$(eval $(kernel-module))
$(eval $(generic-package))
```

Along with the kernel module, we need to enable the CMA, because we only have 128MB to work with I first started with 64MB.

#### defconfig
```diff
+ CONFIG_DMA_CMA=y
+ CONFIG_CMA_SIZE_MBYTES=64
```
#### sun8i.dtsi
```txt
	reserved-memory {
            #address-cells = <1>;
			#size-cells = <1>;
			ranges;

			cma_pool: default-pool {
				alloc-ranges = <0x40000000 0x8000000>; // Start Addr + Length
				compatible = "shared-dma-pool";
				linux,cma-default;
				reusable;
				alignment = <0x2000>;
				size = <0x4000000>; /* 64MB */
			};
	};

    syscon: syscon@1c00000 {
        compatible = "allwinner,sun8i-v3s-system-controller", "allwinner,sun8i-h3-system-control", "syscon";
        reg = <0x01c00000 0xd0>;
        #address-cells = <1>;
        #size-cells = <1>;
        ranges;

        sram_c: sram@4000 {
            compatible = "mmio-sram";
            reg = <0x4000 0xa000>;
            #address-cells = <1>;
            #size-cells = <1>;
            ranges = <0 0x4000 0xa000>;

            ve_sram: sram-section@0 {
                compatible = "allwinner,sun8i-v3s-sram-c", "allwinner,sun4i-a10-sram-c1";
                reg = <0x000000 0xa000>;
            };
        };
    };

    ve: video-engine@01c0e000 {
        compatible = "allwinner,sunxi-cedar-ve";
        reg = <0x01c0e000 0x1000>,
                    <0x01c00000 0x10>,
                    <0x01c20000 0x800>;
        memory-region = <&cma_pool>;
        syscon = <&syscon>;
        clocks = <&ccu CLK_BUS_VE>, <&ccu CLK_VE>,
                    <&ccu CLK_DRAM_VE>;
        clock-names = "ahb", "mod", "ram";
        resets = <&ccu RST_BUS_VE>;
        interrupts = <GIC_SPI 58 IRQ_TYPE_LEVEL_HIGH>;
        allwinner,sram = <&ve_sram 1>;
    };
```

A lot of the references I've seen online to working with this codec seem to set `sram_c` to `0x1d00000`. Which seems to be an empty section of memory, according to the official memory map. So I've set my device tree to point to the `sram_c` defined in the S3 datasheet, this seems to work correctly, atleast for my use cases.


#### Kernel output
```
[    0.000000] Reserved memory: created CMA memory pool at 0x43c00000, size 64 MiB
[    0.000000] OF: reserved mem: initialized node default-pool, compatible id shared-dma-pool
...
[    0.000000] Zone ranges:
[    0.000000]   Normal   [mem 0x0000000040000000-0x0000000047ffffff]
[    0.000000]   HighMem  empty
[    0.000000] Memory: 50136K/131072K available (8192K kernel code, 1478K rwdata, 2800K rodata, 1024K init, 240K bss, 15400K reserved, 65536K cma-reserved, 0K highmem)
...

# modprobe cedar_ve
[    9.202456] cedar_ve: loading out-of-tree module taints kernel.
[    9.210073] sunxi cedar version 0.1 
[    9.213939] [cedar]: install start!!!
[    9.217684] cedar_ve: cedar-ve the get irq is 23
[    9.222323] sunxi-cedar 1c0e000.video-engine: assigned reserved memory node default-pool
[    9.230624] [cedar]: Success claim SRAM
[    9.338704] [cedar]: memory allocated at address PA: 43D00000, VA: C3D00000
[    9.345749] [cedar]: install end!!!
```
Success! 🥳

---

The first time I tried this I hit an error trying to allocate memory, because it was attempting to allocate 80MB. Easy fix. Might be better suited to be set up the ve_size as a device tree parameter.
```diff
diff --git a/cedar_ve.c b/cedar_ve.c
index 2f6f744..c73ea88 100644
--- a/cedar_ve.c
+++ b/cedar_ve.c
@@ -1804,7 +1804,7 @@ static int cedardev_init(struct platform_device *pdev)
 #endif /*LINUX_VERSION_CODE >= KERNEL_VERSION(4,11,0)*/
 
 #if !defined(USE_ION)
-	cedar_devp->ve_size = 80 * SZ_1M;
+	cedar_devp->ve_size = 32 * SZ_1M;
 	ret = dma_set_coherent_mask(cedar_devp->dev, DMA_BIT_MASK(32));
 	if (ret) {
 		dev_err(cedar_devp->dev, "DMA enable failed\n");
```

---

New we have the kernel module loaded it's time to set up FFmpeg.
Here is a diff I've applied to [FFmpeg 4.4](https://github.com/gregdavill/FFmpeg/compare/release/4.4...gregdavill:FFmpeg:cedrus264)

## Results

{{< bundle-image name="images/003.png" >}}

---

Not an apple to oranges comparison, because the h264 parameters are different. But you can see even on the "ultrafast" preset, the speed can't keep up with a measly 30 fps 640x512 stream.

```txt
# ffmpeg -f v4l2 -video_size 640x512 -pix_fmt nv12 -i /dev/video0 -r 30 -c:v cedrus264 -vewait 5 -b:v 4M -maxrate 4M -qp 30 -t 5 -f mp4 test.mp4 -y
frame=  150 fps= 30 q=-0.0 Lsize=     558kB time=00:00:04.96 bitrate= 921.1kbits/s speed=0.994x    
# ffmpeg -f v4l2 -video_size 640x512 -pix_fmt nv12 -i /dev/video0 -r 30 -c:v libx264 -b:v 4M -maxrate 4M -bufsize 1M -preset ultrafast -t 5  -f mp4 test.mp4 -y 
frame=  150 fps= 20 q=7.0 Lsize=    2380kB time=00:00:04.96 bitrate=3925.5kbits/s dup=24 drop=0 speed=0.66x 
```

I've only really started to poke around with the encoder support. But hope this write-up encourages you to give it a spin too! I'd love to see any projects you come up with utilizing the Allwinner S3!
