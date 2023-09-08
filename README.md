`linux-nunchuk` is an embedded Linux system built for the [STM32MP157D-DK1][1] with [Buildroot][2]. It includes a device driver for the Wii Nunchuk.

Buildroot (2023.08) is included as a [git submodule][4]. To clone `linux-nunchuk` and include Buildroot:

```
git clone https://github.com/j0la/linux-nunchuk.git
git submodule update --init
```

This project is based on [Bootlin][3]'s Embedded Linux and Buildroot trainings.

## Hardware

- STM32MP157D-DK1
- 16GB microSD card
- USB Type-C cable
- USB Micro-B cable
- RJ45 cable
- Wii Nunchuk
- Nunchuk [breakout adapter][5]

## Build

`build/` is the out-of-tree destination for Buildroot output. It also contains the `.config`. Run `make menuconfig` to edit the configuration and `make` to build (both from within `build/`).

`build/host` contains the cross-compilation toolchain.

`build/images` contains the bootloader, kernel, and root filesystem images.

## Installation

The U-Boot bootloader, Linux kernel, and root filesystem images must be flashed to the microSD card.

Insert the SD card in the host system and run `dmesg` to find the device name (e.g. `sdb`).

### Create Partitions

First, unmount any mounted partitions:
```
sudo umount /dev/sdb*
```

Clear the first 32MB:
```
sudo dd if=/dev/zero of=/dev/sdb bs=1M count=32
```

Then create partitions for the first and second stage bootloaders, kernel, and filesystem:
```
sudo parted /dev/sdb

GNU Parted 3.4
Using /dev/sdb
Welcome to GNU Parted! Type 'help' to view a list of commands.
(parted) mklabel gpt
(parted) mkpart fsbl1 0% 4095s
(parted) mkpart fsbl2 4096s 6143s
(parted) mkpart ssbl 6144s 10239s
(parted) mkpart boot 10240s 43007s
(parted) mkpart root 43008s 174079s
(parted) toggle 4 boot
(parted) unit kiB
(parted) print

Model: Generic MassStorageClass (scsi)
Disk /dev/sdb: 15558144kiB
Sector size (logical/physical): 512B/512B
Partition Table: gpt
Disk Flags:

Number  Start     End       Size      File system  Name   Flags
 1      1024kiB   2048kiB   1024kiB                fsbl1
 2      2048kiB   3072kiB   1024kiB                fsbl2
 3      3072kiB   5120kiB   2048kiB                ssbl
 4      5120kiB   21504kiB  16384kiB               boot   boot, esp
 5      21504kiB  87040kiB  65536kiB               root
```

Finally, format the boot (kernel) partition as a FAT filesystem:
```
sudo mkfs.vfat -F 32 -n boot /dev/sdb4
```

And the root partition as ext4:
```
sudo mkfs.ext4 -L rootfs -E nodiscard /dev/sdb5
```

### Transfer Images

Eject and re-insert the SD card. `rootfs` should automatically mount, but `boot` may not. Mount it with: `sudo mount /dev/sdb4 /media/$USER/boot`. Check that both partitions are mounted: `lsblk -f`

Flash the bootloader:
```
sudo dd if=build/images/u-boot-spl.stm32 of=/dev/sdb1 bs=1M conv=fdatasync
sudo dd if=build/images/u-boot-spl.stm32 of=/dev/sdb2 bs=1M conv=fdatasync
sudo dd if=build/images/u-boot.img of=/dev/sdb3 bs=1M conv=fdatasync
```

Copy the kernel image and device tree to `boot`:
```
sudo cp build/images/zImage build/images/stm32mp157a-dk1.dtb /media/$USER/boot/
```

Extract the root filesystem to `rootfs`:
```
sudo tar -C /media/$USER/rootfs/ -xf build/images/rootfs.tar
```

Finally, create `/media/$USER/boot/extlinux/extlinux.conf` with the following content to tell U-Boot how to load the kernel:
```
label buildroot
    kernel /zImage
    devicetree /stm32mp157a-dk1.dtb
    append console=ttySTM0,115200 root=/dev/mmcblk0p5 rootwait
```

## Connect to the Board

Insert the SD card and connect the Micro-B cable. Use a serial I/O tool like `tio` to communicate: `tio /dev/ttyACM0`

Power the board with the USB-C cable and you should see the system boot:
```
U-Boot SPL 2022.04 (Sep 08 2023 - 13:01:36 -0500)
...
U-Boot 2022.04 (Sep 08 2023 - 13:01:36 -0500)
...
Starting kernel ...
...
Welcome to Buildroot
buildroot login: root
#
```

[1]: https://www.st.com/en/evaluation-tools/stm32mp157d-dk1.html
[2]: https://buildroot.org/
[3]: https://bootlin.com/training/
[4]: https://git-scm.com/book/en/v2/Git-Tools-Submodules
[5]: https://www.adafruit.com/product/4836