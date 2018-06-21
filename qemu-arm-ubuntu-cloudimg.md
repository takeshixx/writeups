# Running Ubuntu Cloud Images on QEMU ARM

## Prepare cloudinit Configuration Image

```
vagrant@ubuntu-xenial:~/cloudstuff$ cat seed
#cloud-config
password: ubuntu
chpasswd: { expire: False }
ssh_pwauth: True
```

```
sudo docker run -v $(pwd):/build -it ubuntu:latest sh -c "apt update;apt install cloud-image-utils;cloud-localds /build/cloudinit.img /build/seed"
```

A pre-built image with the example configuration is available [here](https://kleber.io/uX5bHfzOjg/) (TODO: add git link as soon as LFS is working again).

## Get the Required Files

```
wget https://cloud-images.ubuntu.com/releases/16.04/release/ubuntu-16.04-server-cloudimg-armhf-disk1.img
wget https://cloud-images.ubuntu.com/releases/16.04/release/unpacked/ubuntu-16.04-server-cloudimg-armhf-vmlinuz-lpae
```

## Boot the ARMHF cloudimg on QEMU

```
qemu-system-arm \
    -m 1024 \
    -M virt \
    -drive if=none,file=ubuntu-16.04-server-cloudimg-armhf-disk1.img,id=hd0 \
    -device virtio-blk-device,drive=hd0 \
    -no-reboot \
    -nographic \
    -monitor telnet:127.0.0.1:44444,server,nowait \
    -serial stdio \
    -kernel ubuntu-16.04-server-cloudimg-armhf-vmlinuz-lpae \
    -append "root=/dev/vda1 rootfstype=ext4" \
    -net nic \
    -net tap,ifname=tap0,script=no,downscript=no \
    -cdrom cloudinit.img
```

*Note*: The `-cdrom` argument is only required on the first boot in order to set the configuration parameters.

A wrapper script for booting the cloudimg images is available [here](https://github.com/takeshixx/tools/blob/master/qemu/qemu_arm_ubuntu_cloudimg.sh).

--


