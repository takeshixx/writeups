# Running Ubuntu 16.04.1 armhf on Qemu

This is a writeup about how to install [Ubuntu 16.04.1 Xenial Xerus](https://wiki.ubuntu.com/XenialXerus/ReleaseNotes) for the 32-bit hard-float ARMv7 (armhf) architecture on a Qemu VM via [Ubuntu netboot](http://cdimage.ubuntu.com/netboot/16.04.1/).

The setup will create a Ubuntu VM with [LPAE](https://www.arm.com/products/processors/technologies/virtualization-extensions.php) extensions (generic-lpae) enabled. However, this writeup should also work for non-LPAE (generic) kernels.

The performance of the resulting VM is quite good, and it allows VMs with >1G ram (compared to 256M on `versatilepb` and 1G on `versatile-a9`/`versatile-a15`). It also supports `virtio` disks whereas `versatile-a9`/`versatile-a15` only support SD cards via the `-sd` argument.

## Get netboot files

The netboot files are available on the official [Ubuntu mirror](http://ports.ubuntu.com/ubuntu-ports/dists/xenial/main/installer-armhf/current/images/generic-lpae/netboot/). The following commands will download the kernel (vmlinuz) and initrd (initrd.gz) in a new directory called `netboot`:

```bash
mkdir netboot
cd netboot
wget -r -nH -nd -np -R "index.html*" --quiet http://ports.ubuntu.com/ubuntu-ports/dists/xenial/main/installer-armhf/current/images/generic-lpae/netboot/
```

## Create a image file

The following command creates a image file that will be used as a disk for the Ubuntu system:

```bash
qemu-img create -f qcow2 ubuntu.img 16G
```

## Setup networking

Installing Ubuntu via netboot requires an Internet connection. Therefore the Ubuntu VM requires a network intefaces that can use the Internet connection of the host system. The host systems needs to create a tuntap device for the VM and a bridge that will connect the tuntap interface with the Internet-facing interface. The bridge is called `br0` and the tuntap device for the Ubuntu VM will be `tap0`.

```bash
sudo ip tuntap add dev tap0 mode tap
sudo ip link set up dev tap0
sudo ip link set tap0 master br0
```

Creating a bridge and adding the Internet-facing interface (e.g. `eth0`):

```bash
sudo ip link add br0 type bridge
sudo ip link set eth0 master br0
```

Allow IP packet forwarding:

```bash
sudo sysctl -w net.ipv4.ip_forward=1
```

Start a DHCP server in the bridge so that the Ubuntu VM receives an IP address:

```bash
echo "subnet 192.168.0.0 netmask 255.255.255.0 {
  range 192.168.0.10 192.168.0.100;
  option routers 192.168.0.1;
  option domain-name-servers 208.67.222.222, 208.67.220.220;
}" > qemu-dhcpd.conf
sudo dhcpd -cf qemu-dhcpd.conf br0
```

Set router IP on bridge interface:

```bash
ip addr add 192.168.0.1/24 dev br0
ip link set up dev br0
```

## Start the netboot installation

From the netboot directory (containing the netboot files `vmlinuz` and `initrd.gz`), run the following command:

```bash
qemu-system-arm \
  -kernel vmlinuz \
  -initrd initrd.gz \
  -append "root=/dev/ram" \
  -no-reboot \
  -nographic \
  -m 1024 \
  -M virt \
  -serial stdio \
  -net nic \
  -net tap,ifname=tap0,script=no,downscript=no \
  -hda ubuntu.img
```

The kernel should now boot into the Ubuntu installer within the terminal window where the command has been executed.

## Extract the new kernel

After the installation process has been finished, extract the kernel and initrd files from the new installed Ubuntu system. Mount the Ubuntu image:

```bash
qemu-img convert -f qcow2 -O raw ubuntu.img ubuntu-raw.img
sudo losetup /dev/loop0 ubuntu-raw.img
OFFSET=$(($(sudo fdisk -l /dev/loop0 |grep /dev/loop0p1 |awk '{print $3}')*512))
sudo mount -o loop,offset=$OFFSET /dev/loop0 /mnt
```

Create a new directory `boot` for the new kernel and initrd files and copy them into it:

```bash
mkdir boot
cp /mnt/initrd.img-4.4.0-38-generic-lpae
cp /mnt/vmlinuz-4.4.0-38-generic-lpae boot
```

Cleanup:

```bash
sudo umount /mnt
sudo losetup -d /dev/loop0
rm ubuntu-raw.img
```

*TODO*: Add working solution that does not require to convert the image file.

## Start the Ubuntu VM

The Ubuntu VM directory should now have the following structure:

```bash
├── boot
│   ├── initrd.img-4.4.0-38-generic-lpae
│   └── vmlinuz-4.4.0-38-generic-lpae
├── netboot
│   ├── initrd.gz
│   └── vmlinuz
└── ubuntu.img
```

Start Qemu:

```bash
qemu-system-arm \
  -kernel boot/vmlinuz-4.4.0-38-generic-lpae \
  -initrd boot/initrd.img-4.4.0-38-generic-lpae \
  -append "root=/dev/vda2 rootfstype=ext4" \
  -no-reboot \
  -nographic \
  -m 1024 \
  -M virt \
  -serial stdio \
  -monitor telnet:127.0.0.1:9000,server,nowait \
  -net nic \
  -net tap,ifname=tap0,script=no,downscript=no \
  -drive file=ubuntu.img,if=virtio
```

The following script can be used for starting the VM:

```bash
#!/bin/sh
NET_IF=tap0
BRIDGE=br0
if ! grep --quiet $NET_IF /proc/net/dev;then
    echo "Creating ${NET_IF} device"
    sudo ip tuntap add dev $NET_IF mode tap
    sudo ip l s up dev $NET_IF
fi
if grep --quiet $BRIDGE /proc/net/dev;then
    sudo ip l s $NET_IF master $BRIDGE
else
    echo "${BRIDGE} does not exist!"
    exit
fi
qemu-system-arm \
  -kernel boot/vmlinuz-4.4.0-38-generic-lpae \
  -initrd boot/initrd.img-4.4.0-38-generic-lpae \
  -append "root=/dev/vda2 rootfstype=ext4" \
  -no-reboot \
  -nographic \
  -m 1024 \
  -M virt \
  -serial stdio \
  -monitor telnet:127.0.0.1:9000,server,nowait \
  -net nic \
  -net tap,ifname=tap0,script=no,downscript=no \
  -drive file=ubuntu.img,if=virtio
```

## Appendix

### Use raw image (by @sokomo)
Just create raw image instead of qcow2 image using command:
```
qemu-img create -f raw ubuntu.img 16G
```

And then you can continue using without any problem
If you get any warning messages about using raw image:
-  In first `qemu-system-arm` command, you can replace the `-hda ubuntu.img` part with `-drive format=raw,file=ubuntu,img,index=0`.
- In second `qemu-system-arm` command, you can replace with `format=raw,file=ubuntu.img,if=virtio` in `-drive` option

Note: To avoid using offset when the raw image has multiple partitions, you can reload the `loop` module:
```
sudo modprobe -r loop
sudo modprobe loop max_loop=10 max_part=15
sudo losetup -f ubuntu.img
sudo mount /dev/loop0p2 /mnt
```
After finishing copying the kernel and initrd file, you can un-mount and detach the image file
```
sudo umount /mnt
sudo losetup -d /dev/loop0
```

### Mount `qcow2` image using `qemu-nbd` (by @sokomo)
Add `nbd` module, and then mount using `qemu-nbd`
```
sudo modprobe nbd max_part=16
sudo qemu-nbd -c /dev/nbd0 ubuntu.img
sudo partprobe /dev/nbd0
sudo mount /dev/nbd0p2 /mnt
```

After finishing copying the kernel and initrd file, you can un-mount and detach the image file
```
sudo umount /mnt/
sudo qemu-nbd -d /dev/nbd0
sudo killall qemu-nbd
```
