# HI3536DV100 SoC

I have Techage N6708G5 8 Channel PoE NVR which uses HI3536DV100 SoC. I'm publishing any information I have about this device.

# Specifications

* CPU - HI3536DV100 single core ARM Cortex-A7 850MHz;
* RAM - 128MB DDR3, although original firmware boots with 83M;
* FLASH - Winbond w25q128 SPI-NOR 16MB;
* Switch - 8+1 Channel PoE switch

# Original firmware

I have dumped the original firmware. You can find mtd block sizes in here as well.

# SDK 

Due some miracle, I was able to retrieve the SDK for this device. It contains kernel, uboot and a few kernel modules sources, many development tutorials regarding the board and it's peripherals and H265/264 library development.

https://mega.nz/#!fegnnCJb!91KLL77uYPvLu-luCYXqPn1txlih-sc-qg0VPm976Lw
https://mega.nz/#!Sfhk3ASS!eTDEElTqV3Glpv9BgOHIsUMXFwjvoVjv7st77Joe5vI

# Obtaining root

It seems that manufacturer patched the available backdoors so the only way to obtain root at the moment is through UART and flashing modified rootfs. On my device, EARLY_PRINTK kernel option is disabled so I don't get any bootup messages after kernel has been started nor received tty console over UART.

In order to modify rootfs, you need to extract `romfs` with `unsquashfs` which contains kernel and basic rootfs with busybox. Then modify `squashfs_romfs/etc/init.d/rcS` file to comment out/remove the following line:

`dvrHelper /lib/modules /usr/bin/Sofia 127.0.0.1 9578 1`

Then make squashfs image with `mksquashfs` and flash through u-boot.

It is needed to remove because `Sofia` application is a daemon and blocks inittab startup once it's launched. The tty console is already defined in `squashfs_romfs/etc/inittab`, but because `Sofia` blocks, tty is never launched.

For UART, you need to do some soldering. I have added the similar picture of my board (will update with my real one soon) in `uart.png`. Baudrate is default - 115200.

# Security

Once root has been obtained I have found out that the switch ports are not isolated, meaning I can't cut off IP camera's traffic even if I install clean linux image on the NVR. It's possible to do some VLAN tagging/isolating, but it needs external re-wiring from PHY device to the HiSi CPU through I2C and writing driver for it. I doubt I will ever tinker with this. One solution I could see is using usb-to-ethernet adapter.

Overall there's nothing much to say about security, it's Chinese software, it's full of bloat/possible malware and unwanted traffic back to homeland (that's why I want to isolate the IP camera's traffic). The NVR runs on 4.9.37 kernel which isn't that old, but still.

# Custom Linux firmware

Most of the work is done for OpenWrt 18.06 Hisilicon HI3536DV100 CPU support. Please see https://github.com/ubis/openwrt/tree/feature/add-hisi35xx-target/target/linux/hisilicon and https://github.com/ubis/openwrt/pull/1

Most hardware is supported and working: Ethernet, SATA, USB, SPI NAND, GPIO(untested), except D-SUB/HDMI/Audio Out. Latter ones are enabled by probing proprietary linux kernel 4.9.37 modules. I do not have sources for them, and OpenWrt 18.06 (as of now 2019-12-28) is using 4.9.207 kernel.

I had plans to fully replace NVR software with OpenWrt & some open source solution, but that seems to be not ideal solution for now.

The reason behind that is mostly motion detection. It requires a right amount of hardware to do, if done on software side. However, these Hisilicon IP Cameras have dedicated hardware for that to reduce CPU usage when doing motion detection. At first I thought that motion detection is configured with ONVIF, since it is supported. Unfortunately once I dig into it further, detection is configured using proprietary protocol and only alarms/triggers are sent by ONVIF messages.

As of now, I see this solution suitable: hook up small ARM board(RPI-like) with dual Ethernet support(could use even usb-ethernet). Fully isolate NVR and it's cameras. Then build rtsp-http streaming service on that ARM board and expose it for further uses. This prevents anything from going out of NVR stuff and only stream is passed.
I could even build a ONVIF client reader on the ARM board and configure it to send email/do something else on trigger event.

# 2021 Update

So I have been running the NVR with 4 cams for a while now, with stock software. I do have a Debian server with Wireguard client in the middle between internet gateway and NVR/CAMs network. NVR/CAMs network doesn't have access to the internet. I have configured to record on motion, however, I noticed that motion detection doesn't always work and I had to switch to 24/7 recording.

I would like to save records 24/7 to my NAS, however, stock sofware (Sofia) allows records to upload only via FTP server and when motion is detected. There is no option for continous record uploading. I haven't figured out what i'm gonna do about it.

Recently I have found 0day vulnerability: https://habr.com/en/post/486856/ and it affected my NVR and CAMs too. I can get root access now without modifying rootfs.
