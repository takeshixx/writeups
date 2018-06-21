# Running Preinstalled Ubuntu 17.04 for Raspi2 on QEMU

Ubuntu releases are always available als pre-installed images for Raspberry Pi 2 and 3. This writeups briefly shows how the Raspberry Pi 2 image can be used to easily and quickly spawn a pre-installed ready-to-use Ubuntu ARM VM on QEMU.

These scripts are available in the [tools](https://github.com/takeshixx/tools) repository.

## Download the Preinstalled Image

```bash
wget http://cdimage.ubuntu.com/ubuntu/releases/17.04/release/ubuntu-17.04-preinstalled-server-armhf+raspi2.img.xz
xd -d ubuntu-17.04-preinstalled-server-armhf+raspi2.img.xz
```

## Setup a Virtual Network

```bash
cd tools/net
./virtual_network.sh eth0
```

*Note*: If your uplink interface is not called `eth0` or if you are using a WIFI uplink, change the external interface.

## Extract the Boot Files

Now you have to extract the boot files (kernel, initrd image and dtb file) from the pre-installed image.

```bash
tools/sys/extract_bootfiles.sh ubuntu-17.04-preinstalled-server-armhf+raspi2.img .
```

## Boot into the System

```bash
tools/qemu/qemu_arm_ubuntu_boot_raspi2.sh \
    vmlinuz \
    initrd.img \
    bcm2709-rpi-2-b.dtb \
    ubuntu-17.04-preinstalled-server-armhf+raspi2.img \
    tap0
```
