# Linux on Microwatt

[Microwatt](https://github.com/antonblanchard/microwatt) is an open source
POWER ISA softcore written in VHDL 2008. It aims to be simple and easy to
understand.

Mainline Linux supports Microwatt as of v5.14.

These instructions are for the Arty A7, which is the most common FPGA dev board
used by Microwatt developers.

There are [other instructions](https://codeconstruct.com.au/docs/microwatt-orangecrab/)
written by Matt from Code Construct for testing on the OrangeCrab.

Microwatt can also be built as part of a LiteX SoC. This configuration is not
as well tested, and is not documented by these ininstructions.

## Programming the board

Clone this repository and run the following steps, or use the instructions
below to build your own artifacts.

1. Program the flash

   This operation will overwrite the contents of your flash.

   For the Arty A7 A100T, set `FLASH_ADDRESS` to `0x400000` and pass `-f a100`.

   For the Arty A7 A35T, set `FLASH_ADDRESS` to `0x300000` and pass `-f a35`.

   For example:

   ```
   openocd/flash-arty -f a100 images/microwatt_0.bit
   openocd/flash-arty -f a100 images/dtbImage.microwatt.elf -t bin -a 0x400000
   ```

2. Connect to the second USB TTY device exposed by the FPGA

   ```
   minicom -D /dev/ttyUSB1
   ```

   The gateware has firmware that will look at `FLASH_ADDRESS` and attempt to
   parse an ELF there, loading it to the address specified in the ELF header
   and jumping to it.

## Building components from source

### Synthesis on Xilinx FPGAs using Vivado

1. Clone microwatt

   ```
   git clone https://github.com/antonblanchard/microwatt
   ```

2. Install Vivado WebPack

   ```
   source /opt/Xilinx/Vivado/2019.1/settings64.sh
   ```

3. Install FuseSoC

   ```
   pip3 install --user -U fusesoc
   fusesoc init
   fusesoc fetch uart16550
   fusesoc library add microwatt /path/to/microwatt
   ```

4. Build gateware using FuseSoC

   First configure FuseSoC as above.

   Replace `arty_a7-100` with `arty_a7-35` if you have an Arty A7 35T.

   ```
   fusesoc run --build --target=arty_a7-100 microwatt --no_bram --memory_size=0
   ```

   The output is `build/microwatt_0/arty_a7-100-vivado/microwatt_0.bit`.

### Software components

### Use buildroot to create a userspace

   A small change is required to glibc in order to support the VMX/AltiVec-less
   Microwatt, as float128 support is mandiatory and for this in GCC requires
   VSX/AltiVec. This change is included in Joel's buildroot fork, along with a
   defconfig:

   Note that this will also build the Linux kernel, so you do not need to build
   it sparely.

   ```
   git clone -b microwatt https://github.com/shenki/buildroot
   cd buildroot
   make ppc64le_microwatt_defconfig
   make
   ```

   The output is `output/images/rootfs.cpio`

   The kernel is `output/images/dtbImage.microwatt.elf`, which embeds a copy of rootfs.cpio.


### Build the Linux kernel

   ```
   git clone https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git
   cd linux
   make ARCH=powerpc microwatt_defconfig
   make ARCH=powerpc CROSS_COMPILE=powerpc64le-linux-gnu- \
     CONFIG_INITRAMFS_SOURCE=/buildroot/output/images/rootfs.cpio -j`nproc`
   ```

   The output is `arch/powerpc/boot/dtbImage.microwatt.elf`.


## Netbooting Linux

Instead of flashing Linux to the SPI NOR, we can load it over the Ethernet
connection on the Arty.

To do this we use a fork of u-boot that supports Microwatt and the LiteEth
peripheral. Follow the instructions above and instead of flashing Linux, flash
u-boot.elf to the same location:

```
   openocd/flash-arty -f $ARTY images/u-boot -t bin -a $FLASH_ADDRESS
```

```
Trying flash...
Copy segment 0 (0x6a020 bytes) to 0xc000000
Booting from DRAM at c000048


U-Boot 2021.04-00924-gc55ea8aac3b6 (Nov 19 2021 - 10:27:41 +0800)

CPU: Microwatt
DRAM:  256 MiB
```

From here, serve the kernel over tftp:

```
sudo apt install python3-fbtftp
sudo python3 /usr/share/doc/python3-fbtftp/examples/server.py --port 69
```

```
=> dhcp
=> setenv serverip 192.168.86.8
=> tftp dtbImage.microwatt.elf
=> bootelf
```

### Building u-boot

To build from source:

```
git clone -b microwatt https://github.com/shenki/u-boot
cd u-boot
make microwatt_arty_defconfig
make CROSS_COMPILE=powerpc64le-linux-gnu- -j`nproc`
```

The output is `u-boot`.
