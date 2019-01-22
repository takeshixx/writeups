# Running Ubuntu 18.04 armhf on QEMU

This is a writeup about how to install [Ubuntu 18.04 Bionic Beaver](https://wiki.ubuntu.com/BionicBeaver/ReleaseNotes) for the 32-bit hard-float ARMv7 (armhf) architecture on a QEMU VM via [Ubuntu netboot](http://cdimage.ubuntu.com/netboot/18.04/).

The setup will create a Ubuntu VM with [LPAE](https://www.arm.com/products/processors/technologies/virtualization-extensions.php) extensions (generic-lpae) enabled. However, this writeup should also work for non-LPAE (generic) kernels.

The performance of the resulting VM is quite good, and it allows VMs with >1G ram (compared to 256M on `versatilepb` and 1G on `versatile-a9`/`versatile-a15`). It also supports `virtio` disks whereas `versatile-a9`/`versatile-a15` only support SD cards via the `-sd` argument.

These scripts are available in the [tools](https://github.com/takeshixx/tools) repository.

## Get netboot files

The netboot files are available on the official [Ubuntu mirror](http://ports.ubuntu.com/ubuntu-ports/dists/bionic/main/installer-armhf/current/images/generic-lpae/netboot/). The following commands will download the kernel (vmlinuz) and initrd (initrd.gz) in a new directory called `netboot`:

```bash
mkdir netboot
cd netboot
wget -r -nH -nd -np -R "index.html*" --quiet http://ports.ubuntu.com/ubuntu-ports/dists/bionic/main/installer-armhf/current/images/generic-lpae/netboot/
```

## Create a image file

The following command creates a image file that will be used as a disk for the Ubuntu system:

```bash
qemu-img create -f qcow2 ubuntu-bionic.img 16G
```

## Setup networking

Installing Ubuntu via netboot requires an Internet connection. Therefore the Ubuntu VM requires a network intefaces that can use the Internet connection of the host system. The host systems needs to create a tuntap device for the VM and a bridge that will connect the tuntap interface with the Internet-facing interface. The bridge is called `br999` and the tuntap device for the Ubuntu VM will be `tap0`.

```bash
tools/net/virtual_network.sh eth0
```

*Note*: If your uplink interface is not called `eth0` or if you are using a WIFI uplink, change the external interface.
*Note*: For manual network setups, see this.

## Start the netboot installation

From the netboot directory (containing the netboot files `vmlinuz` and `initrd.gz`), run the following command:

```bash
tools/qemu/qemu_arm_ubuntu_netboot.sh \
    netboot/vmlinuz \
    netboot/initrd.gz \
    ubuntu-bionic.img \
    tap0
```

The kernel should now boot into the Ubuntu installer within the terminal window where the command has been executed.

## Extract the Boot Files

Now you have to extract the boot files (kernel and initrd image) from the freshly installed image.

```bash
mkdir boot
tools/sys/extract_bootfiles.sh ubuntu-bionic.img boot
```

## Start the Ubuntu VM

```bash
tools/qemu/qemu_arm_ubuntu_boot.sh \
    boot/vmlinuz \
    boot/initrd.img \
    ubuntu-bionic.img \
    tap0
```
