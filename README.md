# HI3536DV100 SoC

I have Techage N6708G5 8 Channel PoE NVR which uses HI3536DV100 SoC. I'm publishing any information I have about this device.

# Specifications

* CPU - HI3536DV100 single core ARM Cortex-A7 850MHz;
* RAM - 128MB DDR3, although original firmware boots with 83M;
* FLASH - Winbond 25q128 SPI-NOR 16MB;
* Switch - 8+1 Channel PoE switch

# Original firmware

I have dumped the original firmware. You can find mtd block sizes in here as well.

# SDK 

Due some miracle, I was able to retrieve the SDK for this device. It contains kernel, uboot and a few kernel modules sources, many development tutorials regarding the board and it's peripherals and H265/264 library development.

# Obtaining root

It seems that manufacturer patched the available backdoors so the only way to obtain root at the moment is through UART and flashing modified rootfs. On my device, EARLY_PRINTK kernel option is disabled so I don't get any bootup messages after kernel has been started nor received tty console over UART.

In order to modify rootfs, you need to extract `romfs` with `unsquashfs` which contains kernel and basic rootfs with busybox. Then modify `squashfs_romfs/etc/init.d/rcS` file to comment out/remove the following line:

`dvrHelper /lib/modules /usr/bin/Sofia 127.0.0.1 9578 1`

Then make squashfs image with `mksquashfs` and flash through u-boot.

It is needed to remove because `Sofia` application is a daemon and blocks inittab startup once it's launched. The tty console is already defined in `squashfs_romfs/etc/inittab`, but because `Sofia` blocks, tty is never launched.

For UART, you need to do some soldering. I have added the similar picture of my board (will update with my real one soon) in `uart.png`. Baudrate is default - 115200.

# Security

Once root has been obtained I have found out that the switch ports are not isolated, meaning I can't cut off IP camera's traffic even if I install clean linux image on the NVR. It's possible to do some VLAN tagging/isolating, but it needs external re-wiring from PHY device to the HiSi CPU through I2C and writing driver for it. I doubt I will ever tinker with this. One solution I could see is using usb-to-ethernet adapter.

Overall there's nothing much to say about security, it's Chinese software, it's full of bloat/possible malware and unwanted traffic back to homeland (that's why I want to isolate the IP camera's traffic). The NVR runs on 4.9.36 kernel which isn't that old, but still.

# Custom Linux firmware

In progress with OpenWRT 18.06 porting. Sources are in https://github.com/ubis/openwrt/tree/feature/add-hisi35xx-target/target/linux/hisilicon

Currently, only required patches are added to the target - CPU support and SPI-NOR support. Bootable with or without initramfs. At the moment, staying on the latest OpenWRT supported 4.9.x kernel. Will switch to 4.14 probably once other patches will be done.

My plan is to make OpenWRT fully supported, then use something like https://shinobi.video/ NVR software solution. Although the latter contains too much bloat - running on NodeJS, which is overkill for my 16MB flash. Probably I will rewrite it in C, after all, it's main component is ffmpeg.
 
I don't believe that video motion detection is done on the NVR device, since it doesn't have such powerful CPU. I think it's done on the each camera, if configured. This way it would make sense how such tiny NVR could handle up to 1080p 8 channel IP cameras. So I don't have any plans to implement motion detection on the NVR, because of not enough power.

Now regarding kernel modules... Some of them are published with source in the SDK, some aren't. I have tried to reverse-engineer one of the modules and it seems it can be done, since those modules doesn't contain complex stuff. However, this is not my main priority, because NVR runs fine without modules right now. I might come back to this once I'll done porting OpenWRT.
