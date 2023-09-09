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

`cfg/` is the br2-external Buildroot tree. It contains project configuration and package recipes.

`build/` is the out-of-tree destination for Buildroot output.

For the first build, run:
```
linux-nunchuk/buildroot/ $ make BR2_EXTERNAL=../cfg O=../build menuconfig
```

Load the configuration:
```
linux-nunchuk/build/ $ make stm32mp157d-dk1_defconfig
```

From now on, use:
```
linux-nunchuk/build/ $ make menuconfig
linux-nunchuk/build/ $ make
```

To update the defconfig (`cfg/configs/stm32mp157d-dk1_defconfig`):
```
linux-nunchuk/build/ $ make savedefconfig
```

## Install

Insert the SD card in the host system and run `dmesg` to find the device name (e.g. `sdb`).

### Create Partitions

First, unmount any mounted partitions:
```
sudo umount /dev/sdb*
```

Clear the begnning of the card:
```
sudo dd if=/dev/zero of=/dev/sdb bs=1M count=32
```

Then create partitions for the first and second stage bootloaders (fsbl1, fsbl2, ssbl), and the root filesystem (root):
```
$ sudo parted /dev/sdb

GNU Parted 3.4
Using /dev/sdb
Welcome to GNU Parted! Type 'help' to view a list of commands.
(parted) mklabel gpt
(parted) mkpart fsbl1 0% 4095s
(parted) mkpart fsbl2 4096s 6143s
(parted) mkpart ssbl 6144s 10239s
(parted) mkpart root 10240s 141311s
(parted) toggle 4 boot
(parted) unit kiB
(parted) print

Model: Generic MassStorageClass (scsi)
Disk /dev/sdb: 15.9GB
Sector size (logical/physical): 512B/512B
Partition Table: gpt
Disk Flags: 

Number  Start   End     Size    File system  Name   Flags
 1      1049kB  2097kB  1049kB               fsbl1
 2      2097kB  3146kB  1049kB               fsbl2
 3      3146kB  5243kB  2097kB               ssbl
 4      5243kB  72.4MB  67.1MB               root   boot, esp
```

Finally, format the root partition as an ext4 filesystem:
```
sudo mkfs.ext4 -L rootfs -E nodiscard /dev/sdb4
```

### Transfer Images

Eject and re-insert the SD card. Check if `rootfs` is mounted (`lsblk -f`) and if not, mount it: `sudo mount /dev/sdb4 /mnt`.

Flash the bootloader:
```
sudo dd if=build/images/u-boot-spl.stm32 of=/dev/sdb1 bs=1M conv=fdatasync
sudo dd if=build/images/u-boot-spl.stm32 of=/dev/sdb2 bs=1M conv=fdatasync
sudo dd if=build/images/u-boot.img of=/dev/sdb3 bs=1M conv=fdatasync
```

Extract the root filesystem to `rootfs`:
```
sudo tar -C /mnt -xf build/images/rootfs.tar
```

The kernel image and device tree are installed in the target's `/boot` and loaded by U-Boot using `/boot/extlinux/extlinux.conf`.

## Connect

### Serial I/O

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

### Network

The target IP address (192.168.0.2) is configured in `/etc/network/interfaces`.

Find the host's ethernet interface name:
```
$ nmcli device status

DEVICE             TYPE      STATE         CONNECTION    
enx4865ee175e87    ethernet  connected     ethernet-enx4865ee175e87
```

Then configure a host IP address:
```
nmcli con add type ethernet ifname enx4865ee175e87 ip4 192.168.0.1/24
```

Test the connection:
```
$ ping 192.168.0.2

PING 192.168.0.2 (192.168.0.2) 56(84) bytes of data.
64 bytes from 192.168.0.2: icmp_seq=1 ttl=64 time=1.88 ms
...
```

### SSH

`dropbear` is included in the Buildroot configuration. Generate an SSH key pair and copy the public key to `cfg/board/rootfs-overlay/root/.ssh/authorized_keys`. Connect with `ssh root@192.168.0.2`.

## Nunchuk

`nunchuk/` contains Bootlin's Nunchuk device driver. `cfg/package/nunchuk-driver` is the Buildroot package.

The custom device tree (`cfg/board/stm32mp157d-dk1.dts`) defines the Nunchuk on the I2C5 bus. Connect the Nunchuk to I2C5 according to the [pinout][6] (pins 3 and 5 on CN2). The device should show up at address 0x52:
```
# i2cdetect -y 1
     0  1  2  3  4  5  6  7  8  9  a  b  c  d  e  f
00:          -- -- -- -- -- -- -- -- -- -- -- -- -- 
10: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- 
20: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- 
30: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- 
40: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- 
50: -- -- 52 -- -- -- -- -- -- -- -- -- -- -- -- -- 
60: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- 
70: -- -- -- -- -- -- -- --  
```

Verify that the nunchuk driver can be loaded and finds the device:
```
# modprobe nunchuk
[  144.047034] input: Wii Nunchuk as /devices/platform/soc/40015000.i2c/i2c-1/1-0052/input/input2
[  144.055898] Nunchuk device probed successfully
```

Use `evtest` to watch input events:
```
# evtest
No device specified, trying to scan all of /dev/input/event*
Available devices:
...
/dev/input/event1:	Wii Nunchuk
Select the device event number [0-1]: 1
```

[1]: https://www.st.com/en/evaluation-tools/stm32mp157d-dk1.html
[2]: https://buildroot.org/
[3]: https://bootlin.com/training/
[4]: https://git-scm.com/book/en/v2/Git-Tools-Submodules
[5]: https://www.adafruit.com/product/4836
[6]: https://wiki.stmicroelectronics.cn/stm32mpu/wiki/STM32MP157x-DKx_-_hardware_description#GPIO_mapping