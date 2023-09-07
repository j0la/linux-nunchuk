`linux-nunchuk` is an embedded Linux system built for the [STM32MP157D-DK1][1] with [Buildroot][2]. It includes a device driver for the Wii Nunchuk from [Bootlin's Embedded Linux training][3].

Buildroot (2023.08) is included as a [git submodule][4]. To clone `linux-nunchuk` and include Buildroot:

```
git clone https://github.com/j0la/linux-nunchuk.git
git submodule update --init
```

`build/` is the out-of-tree destination for Buildroot output. It also contains the `.config`. Run `make menuconfig` and `make` from within this directory.

[1]: https://www.st.com/en/evaluation-tools/stm32mp157d-dk1.html
[2]: https://buildroot.org/
[3]: https://bootlin.com/training/embedded-linux/
[4]: https://git-scm.com/book/en/v2/Git-Tools-Submodules