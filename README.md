# LCA2021

## Presentation Slide

Getting started with LinuxBoot Firmware on AArch64 Server

## How to boot CentOS 8.2 AArch64 from LinuxBoot Firmware using QEMU

```
ubuntu@bionic:~$ git clone https://github.com/NaohiroTamura/LCA2021

ubuntu@bionic:~$ cd LCA2021
```

### Create AArch64 OVMF 32MB Firmware File System

```
ubuntu@bionic:~/LCA2021[master]$ git clone https://github.com/tianocore/edk2

ubuntu@bionic:~/LCA2021[master]$ cd edk2

ubuntu@bionic:~/LCA2021/edk2[master]$ git remote add lca2021 https://github.com/NaohiroTamura/edk2.git

ubuntu@bionic:~/LCA2021/edk2[master]$ git fetch lca2021 aarch64-flashrom

ubuntu@bionic:~/LCA2021/edk2[aarch64-flashrom]$ git checkout aarch64-flashrom

ubuntu@bionic:~/LCA2021/edk2[aarch64-flashrom]$ git submodule update --init

ubuntu@bionic:~/LCA2021/edk2[aarch64-flashrom]$ make -C BaseTools

ubuntu@bionic:~/LCA2021/edk2[aarch64-flashrom]$ . edksetup.sh

ubuntu@bionic:~/LCA2021/edk2[aarch64-flashrom]$ aarch64-linux-gnu-gcc --version
aarch64-linux-gnu-gcc (Ubuntu/Linaro 7.5.0-3ubuntu1~18.04) 7.5.0

ubuntu@bionic:~/LCA2021/edk2[aarch64-flashrom]$ GCC5_AARCH64_PREFIX=aarch64-linux-gnu- build -a AARCH64 -t GCC5 -p ArmVirtPkg/ArmVirtQemu.dsc

ubuntu@bionic:~/LCA2021/edk2[aarch64-flashrom]$ ls -lh Build/ArmVirtQemu-AARCH64/DEBUG_GCC5/FV/*.fd
-rw-rw-r-- 1 ubuntu ubuntu  32M Dec 21 07:15 Build/ArmVirtQemu-AARCH64/DEBUG_GCC5/FV/QEMU_EFI.fd
-rw-rw-r-- 1 ubuntu ubuntu 768K Dec 21 07:15 Build/ArmVirtQemu-AARCH64/DEBUG_GCC5/FV/QEMU_VARS.fd

ubuntu@bionic:~/LCA2021/edk2[aarch64-flashrom]$ cd ~
```

### Configure LinuxBoot Kernel and Initramfs

```
ubuntu@bionic:~$ export GOPATH=~/LCA2021/go

ubuntu@bionic:~$ export PATH=${GOPATH}/bin:/opt/go/bin:$PATH

ubuntu@bionic:~$ go version
go version go1.13.11 linux/amd64

ubuntu@bionic:~$ go get -u github.com/u-root/u-root

ubuntu@bionic:~$ cd ~/LCA2021/go/src/github.com/u-root/u-root/

ubuntu@bionic:~/LCA2021/go/src/github.com/u-root/u-root[master]$ git remote add lca2021 https://github.com/NaohiroTamura/u-root.git

ubuntu@bionic:~/LCA2021/go/src/github.com/u-root/u-root[master]$ git fetch lca2021 centos8-bls-support

ubuntu@bionic:~/LCA2021/go/src/github.com/u-root/u-root[master]$ git checkout centos8-bls-support

ubuntu@bionic:~/LCA2021/go/src/github.com/u-root/u-root[centos8-bls-support]$ go install github.com/u-root/u-root

ubuntu@bionic:~/LCA2021/go/src/github.com/u-root/u-root[centos8-bls-support]$ cd ~/LCA2021

ubuntu@bionic:~/LCA2021[master]$ GOARCH=arm64 u-root -build=bb -o=initramfs.linux_arm64.cpio -uinitcmd=boot core github.com/u-root/u-root/cmds/boot/boot

ubuntu@bionic:~/LCA2021[master]$ xz --check=crc32 -9 --lzma2=dict=1MiB --stdout initramfs.linux_arm64.cpio | dd conv=sync bs=512 of=initramfs.linux_arm64.cpio.xz

ubuntu@bionic:~/LCA2021[master]$ ls -lh initramfs.linux_arm64.cpio*
-rw-r--r-- 1 ubuntu ubuntu  14M Dec 21 08:38 initramfs.linux_arm64.cpio
-rw-rw-r-- 1 ubuntu ubuntu 3.5M Dec 21 08:39 initramfs.linux_arm64.cpio.xz
```

```
ubuntu@bionic:~/LCA2021[master]$ wget https://cdn.kernel.org/pub/linux/kernel/v5.x/linux-5.9.15.tar.xz

ubuntu@bionic:~/LCA2021[master]$ tar xvf linux-5.9.15.tar.xz

ubuntu@bionic:~/LCA2021[master]$ mkdir build-5.9.15

ubuntu@bionic:~/LCA2021[master]$ cp linuxboot-5.9.0-aarch64_defconfig linux-5.9.15/arch/arm64/configs

ubuntu@bionic:~/LCA2021[master]$ cd linux-5.9.15

ubuntu@bionic:~/LCA2021/linux-5.9.15[master]$ ARCH=arm64 CROSS_COMPILE=/usr/bin/aarch64-linux-gnu- make linuxboot-5.9.0-aarch64_defconfig O=../build-5.9.15

ubuntu@bionic:~/LCA2021/linux-5.9.15[master]$ ARCH=arm64 CROSS_COMPILE=/usr/bin/aarch64-linux-gnu- make -j4 O=../build-5.9.15

ubuntu@bionic:~/LCA2021/linux-5.9.15[master]$ ls -lh ../build-5.9.15/arch/arm64/boot/Image*
-rw-rw-r-- 1 ubuntu ubuntu  15M Dec 21 09:30 ../build-5.9.15/arch/arm64/boot/Image
-rw-rw-r-- 1 ubuntu ubuntu 8.2M Dec 21 09:30 ../build-5.9.15/arch/arm64/boot/Image.gz

ubuntu@bionic:~/LCA2021/linux-5.9.15[master]$ cd ~/LCA2021
```

### Inject LinuxBoot into AArch64 64MB Flashrom

```
ubuntu@bionic:~/LCA2021[master]$ dd of=QEMU_EFI-pflash.raw if=/dev/zero bs=1M count=64

ubuntu@bionic:~/LCA2021[master]$ dd of=QEMU_EFI-pflash.raw if=edk2/Build/ArmVirtQemu-AARCH64/DEBUG_GCC5/FV/QEMU_EFI.fd conv=notrunc

ubuntu@bionic:~/LCA2021[master]$ dd of=vars-template-pflash.raw if=/dev/zero bs=1M count=64

ubuntu@bionic:~/LCA2021[master]$ ls -lh *.raw
-rw-rw-r-- 1 ubuntu ubuntu 64M Dec 21 14:03 QEMU_EFI-pflash.raw
-rw-rw-r-- 1 ubuntu ubuntu 64M Dec 21 14:05 vars-template-pflash.raw

ubuntu@bionic:~/LCA2021[master]$ go get -u github.com/linuxboot/fiano/cmds/utk

ubuntu@bionic:~/LCA2021[master]$ utk QEMU_EFI-pflash.raw replace_pe32 Shell build-5.9.15/arch/arm64/boot/Image save QEMU_EFI-pflash-linux.raw 

ubuntu@bionic:~/LCA2021[master]$ ls -lh QEMU_EFI-pflash-linux.raw
-rw-rw-r-- 1 ubuntu ubuntu 64M Dec 21 14:35 QEMU_EFI-pflash-linux.raw
```

### Boot Final OS from Local Disk

```
ubuntu@bionic:~/LCA2021[master]$ /opt/qemu-5.1.0/bin/qemu-system-aarch64 -m 8192 \
-drive if=pflash,format=raw,readonly,file=QEMU_EFI-pflash-linux.raw \
-drive if=pflash,format=raw,file=vars-template-pflash.raw \
-device virtio-rng-pci -nographic -serial mon:stdio \
-machine virt,accel=tcg -cpu cortex-a72 \
-hda centos8-aarch64-lvm.qcow2
```

### Debug LinuxBoot AArch64 Kernel using QEMU and GDB on x86_64

```
ubuntu@bionic:~/LCA2021[master]$ /opt/qemu-5.1.0/bin/qemu-system-aarch64 -s -S -m 8192 \
-drive if=pflash,format=raw,readonly,file=QEMU_EFI-pflash-linux.raw \
-drive if=pflash,format=raw,file=vars-template-pflash.raw \
-device virtio-rng-pci -nographic -serial mon:stdio \
-machine virt,accel=tcg -cpu cortex-a72 \
-hda centos8-aarch64-lvm.qcow2
```

```
ubuntu@bionic:~/LCA2021[master]$ cat ~/.gdbinit
set auto-load safe-path /

ubuntu@bionic:~/LCA2021[master]$ /opt/gdb-9.2/bin/aarch64-gnu-linux-gnu-gdb build-5.9.15/vmlinux
GNU gdb (GDB) 9.2
...
For help, type "help".
Type "apropos word" to search for commands related to "word"...
Reading symbols from build-5.9.15/vmlinux...
(gdb) target remote :1234
Remote debugging using :1234
0x0000000000000000 in ?? ()
(gdb) b start_kernel
Breakpoint 1 at 0xfffffe0010990da4: file /home/ubuntu/LCA2021/linux-5.9.15/init/main.c, line 847.
(gdb) c
Continuing.

Breakpoint 1, start_kernel () at /home/ubuntu/LCA2021/linux-5.9.15/init/main.c:847
847     {
(gdb)
```
