# Orange PI Zero 3 bootloader build instructions
This repo contains instruction on how to build BL (ATF + U-Boot) for
relatively cheap and quite competent board powered by H618 SoC - Orange PI Zero 3.

The result of those instructions has been tested on 1GB variant of the board.
For things I've tested, the Ethernet was working just fine along with
tftp (only while connected point to point, I had some troubles to get it working
through router/switch), SD card and USB.

## Prerequisites
- Linux machine to perform the build (tested on Ubuntu server 20.04.6 LTS)
- Some cross compiler for aarch64. This one should cut:
  https://developer.arm.com/-/media/Files/downloads/gnu-a/10.3-2021.07/binrel/gcc-arm-10.3-2021.07-x86_64-aarch64-none-linux-gnu.tar.xz
  Or install one via package manager eg. for ubuntu: 
  ```
  sudo apt update
  sudo apt -y install gcc-aarch64-linux-gnu
  ```
- SD card to flash the bootloader

## Build instruction

### ATF
All points below must be done in a single instance of the terminal.
- Unset variable so build won't fail (got caught by this - 
  the only reason I'm mentioning this):
```
unset BL31
```
- Clone the ATF sources, you can also use
  ```trusted-firmware-a-88b2d81345dfd84902aae586a743d00ac5df2f48.tar.gz```
  from ```src-mirror``` directory in this repo:
```
git clone https://git.trustedfirmware.org/TF-A/trusted-firmware-a.git
cd trusted-firmware-a && git checkout 88b2d81345dfd84902aae586a743d00ac5df2f48
```
- Proceed with the build itself (in cloned ```trusted-firmware-a``` directory):
```
make CROSS_COMPILE=aarch64-linux-gnu- PLAT=sun50i_h616 DEBUG=1
```
- Export the BL31 variable, so u-boot can assemble the final BL image:
```
export BL31=$(pwd)/build/sun50i_h616/debug/bl31.bin
```

### U-Boot
All points below must be done in a single instance of the terminal, which,
in addition has properly exported variable ```BL31```. See last point in ATF
section.

- Clone the U-Boot sources, you can also use
```u-boot-master-252592214f79d8206c3cf0056a8827a0010214e0.zip```
from ```src-mirror``` directory in this repo:
```
git clone https://github.com/u-boot/u-boot.git
cd u-boot && git checkout 252592214f79d8206c3cf0056a8827a0010214e0
```
- Copy the patch file from this repo to ```u-boot``` directory and apply it
  while still being in ```u-boot``` directory.
  The patch adds defconfig for Orange PI Zero 3, proper DTS, DDR4 DRAM,
  and PMIC support (not sure if the latter is fully functional, never needed it):
```
git apply 0001-Defconfig-For-Orangepi-Zero3-PMIC-Env-From-Fat.patch
```
or alternitavely with patch:
```
patch -p1 < 0001-Defconfig-For-Orangepi-Zero3-PMIC-Env-From-Fat.patch
```
- Generate ```.config``` by utilizing defconfig for Zero3 board, invoke
  (in cloned ```u-boot``` directory):
```
CROSS_COMPILE=aarch64-linux-gnu- make orangepi_zero3_defconfig 
```
- Proceed with the build itself, invoke (in cloned ```u-boot``` directory):
```
CROSS_COMPILE=aarch64-linux-gnu- make -j $((nproc+2)) CONFIG_SPL_IMAGE_TYPE=sunxi_egon
```

This will produce a combined ATF+U-Boot bootloader image ready to be flashed
to SD card. The image can be found in ```u-boot``` directory as ```u-boot-sunxi-with-spl.bin```.

## Flashing to SD card
You can flash the bootloader image ```u-boot-sunxi-with-spl.bin``` directly to
SD card at offset 0x2000 by invoking.
```
sudo dd if=u-boot-sunxi-with-spl.bin of=/dev/<sd card dev> bs=1024 seek=8
```
eg.
```
sudo dd if=u-boot-sunxi-with-spl.bin of=/dev/sdb bs=1024 seek=8
```

If the card is properly formatted you can have both FAT32 partition
along with the bootloader flashed into card.

Having the FAT32 partition is advised, since U-Boot built here stores its
environment there by default. This can be done rootless by mtools package, as
described below.

### U-Boot with FAT32 partition
Reference SD card image can be found in archive ```orangepi-zero3.sdcard.7z``` in
this repo. Remember to un7zip it before flashing to sd card!
- Install mtools:
```
sudo apt update
sudo apt -y install mtools
```
- Create zeroed 256 MiB file (can be larger if you wish to):
```
dd if=/dev/zero of=orangepi-zero3.sdcard bs=1M count=256
```
- Format it with FAT32 by using mtools:
```
mformat -F -i orangepi-zero3.sdcard
```
- Move the FAT a bit further (to 2MiB offset) so we fit BL in between disk metadata and the FAT itself:
```
dd if=orangepi-zero3.sdcard of=orangepi-zero3.sdcard bs=1 skip=$(($(hexdump -s 11 -n 2 -e '"%u"' ./orangepi-zero3.sdcard)*$(hexdump -s 14 -n 2 -e '"%u"' ./orangepi-zero3.sdcard))) seek=2097152 count=16 conv=notrunc

echo -e -n '\x00\x02\x01\x00\x10' | dd if=/dev/stdin of=orangepi-zero3.sdcard bs=1 seek=11 conv=notrunc
```
- Flash the bootloader to altered card image:
```
dd if=u-boot-sunxi-with-spl.bin of=orangepi-zero3.sdcard bs=1024 seek=8 conv=notrunc
```
- Now you can simply flash the SD Card with above image like so 
  (or the one from ```orangepi-zero3.sdcard.7z``` after un7zipping it first):
```
sudo dd if=orangepi-zero3.sdcard of=/dev/<sd card dev> bs=1M
```
eg.
```
sudo dd if=orangepi-zero3.sdcard of=/dev/sdb bs=1M
```

The board should boot right away and print bootlog to ```DEBUG TTL UART``` (the one next to SIP3 label)
with boudrate 115200. See image to find the proper UART port:
http://www.orangepi.org/img/zero3/0627-zero3%20(6).png

Reference bootlog:
```
U-Boot SPL 2023.10-rc4-00039-g252592214f-dirty (Oct 24 2023 - 21:36:06 +0000)
DRAM: 1024 MiB
Trying to boot from MMC1
NOTICE:  BL31: v2.9(debug):v2.9.0-660-g88b2d8134
NOTICE:  BL31: Built : 21:29:55, Oct 24 2023
NOTICE:  BL31: Detected Allwinner H616 SoC (1823)
NOTICE:  BL31: Found U-Boot DTB at 0x4a0b2a38, model: OrangePi Zero3
INFO:    ARM GICv2 driver initialized
INFO:    Configuring SPC Controller
INFO:    PMIC: Probing AXP305 on RSB
ERROR:   RSB: set run-time address: 0x10003
INFO:    Could not init RSB: -65539
INFO:    BL31: Platform setup done
INFO:    BL31: Initializing runtime services
INFO:    BL31: cortex_a53: CPU workaround for erratum 855873 was applied
INFO:    BL31: cortex_a53: CPU workaround for erratum 1530924 was applied
INFO:    PSCI: Suspend is unavailable
INFO:    BL31: Preparing for EL3 exit to normal world
INFO:    Entry point address = 0x4a000000
INFO:    SPSR = 0x3c9
INFO:    Changed devicetree.


U-Boot 2023.10-rc4-00039-g252592214f-dirty (Oct 24 2023 - 21:36:06 +0000) Allwinner Technology

CPU:   Allwinner H616 (SUN50I)
Model: OrangePi Zero3
DRAM:  1 GiB
Core:  53 devices, 22 uclasses, devicetree: separate
WDT:   Not starting watchdog@30090a0
MMC:   mmc@4020000: 0
Loading Environment from FAT... Unable to read "uboot.env" from mmc0:1...
In:    serial@5000000
Out:   serial@5000000
Err:   serial@5000000
Allwinner mUSB OTG (Peripheral)
Net:   eth0: ethernet@5020000using musb-hdrc, OUT ep1out IN ep1in STATUS ep2in
MAC de:ad:be:ef:00:01
HOST MAC de:ad:be:ef:00:00
RNDIS ready
, eth1: usb_ether
Hit any key to stop autoboot:  0
=>
```