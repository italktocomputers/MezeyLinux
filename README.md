# 🪐 Mezey GNU/Linux for Raspberry PI 4 (ARM64)

| Author | Date of Publication | Date of Modification | Mailing list
--- | --- | ---| ---|
|Andrew Schools|Feb 27th, 2021 |Sep 29th, 2022|[📧](https://groups.google.com/g/mezey-linux)

> **Note**
> 
> This documentation is not complete!

Index
-----

*   [Intro](#intro)
*   [Why Mezey GNU/Linux](#why-mezey-gnulinux)
*   [Install guide](#install-guide)
*   [Build Computer Dependencies](#build-computer-dependencies)
*   [Clean Environment](#clean-environment)
*   [Global variables](#global-variables)
*   [Partition a new hard drive](#partition-a-new-hard-drive)
*   [Mount new partition](#mount-new-partition)
*   [Mezey directory tree](#mezey-directory-tree)
*   [Get the Linux Kernel headers](#get-the-linux-kernel-headers)
*   [GNU Binutils (Pass 1)](#gnu-binutils-pass-1)
*   [GCC, the GNU Compiler Collection Intro](#gcc-the-gnu-compiler-collection-intro)
*   [Dowloading GCC, the GNU Compiler Collection Intro](#downloading-the-gcc-the-gnu-compiler-collection)
*   [Downloading GMP/MPFR/MPC](#downloading-gmp-mpfr-mpc)
*   [Building GCC, the GNU Compiler Collection](#building-gcc-the-gnu-compiler-collection)
*   [GNU](#gnu-c-library)
*   [Setup GNU BASH](#setup-gnu-bash)
*   [Setup GNU Coreutils](#setup-gnu-coreutils)
*   [Compile the Linux Kernel](#compile-the-linux-kernel)
*   [Install Linux Kernel and its drivers](#install-linux-kernel-and-its-drivers)
*   [Create a boot loader](#create-a-boot-loader)
*   [Verify Mezey GNU/Linux boot directory](#verify-mezey-gnulinux-boot-directory)
*   [Simple test init](#simple-test-init)
*   [Setup our real init system](#setup-our-real-init-system)
*   [Using Chroot](#using-chroot)
*   [User Setup](#user-setup)
*   [Setup Networking](#setup-networking)
*   [Create an ISO Image](#create-an-iso-image)
*   [Setting up the mezpkg tool](#setting-up-the-mezpkg-tool)
*   [Anatomy of a mezpkg file](#anatomy-of-a-mezpkg-file)
*   [Installing mezpkg manager](#installing-mezpkg-manager)
*   [How to use mezpkg manager](#how-to-use-mezpkg-manager)
*   [Notes](#notes)
*   [Known issues](#known-issues)
*   [Trouble shooting](#trouble-shooting)

Intro
-----

Before proceeding, understand that Mezey GNU/Linux isn't meant for a first time Linux user. If you just want a Linux distribution that will work out of the box after running a fancy installer, I recommend something like Ubuntu or Fedora.

Be warned, this document may be rough around the edges. I may be quickly glossing over things I assume you, the reader already understands. If you think I missed something, or see something wrong with this document, please reach out to me here: [https://groups.google.com/g/mezey-linux](https://groups.google.com/g/mezey-linux)

One last thing. This guide is for Mezey GNU/Linux ARM64 running on a Raspberry PI 4 computer, however, I will be building Mezey GNU/Linux on a AMD64 (x86\_64) system so we will be dealing with cross-compilation. I'm doing this because my build computer has an 2.4 GHz 8-Core Intel Core i9 processor and 32 GB of RAM, compared to the host computer which is using a Broadcom BCM2711, Quad core Cortex-A72 (ARM v8) 64-bit SoC @ 1.5GHz with only 8 GB of RAM. I will be creating a guide on how to build Mezey GNU/Linux using Mezey GNU/Linux (running on a Raspberry PI 4), though this will be a much slower build, and will require packaging tools used for the build process.

[Back to top](#top)

Why Mezey GNU/Linux
-------------------

Mezey GNU/Linux may be for you if you are someone who really wants to understand how the Linux Kernel and the GNU tools that make up a Linux distribution works, or you like the complete flexibility of compiling everything you need to use with the absolute freedom in what goes into your Linux build.

Mezey GNU/Linux does have its own package manager, but these are just packages I have built for my system. If you need software outside of that realm, you need to compile it and install it yourself. If that software has dependencies, you need to also compile and install them. If you need security updates, you need to join some mailing lists. More on the mezeypkg file format and the Mezey package manager later.

[Back to top](#top)

Install guide
-------------

### Build Computer Dependencies

> **Note** 
> 
> To follow along with this guide, you need to be running a version of Linux x86\_64 (amd64) before proceeding. I am using Ubuntu 18.04 but the flavor of Linux doesn't really matter as long as you meet the following requirements below. If you are using a different package manager other than APT, some of the package names below will be named differently.

Required packages (Ubuntu 18.04):

*   make
*   gcc
*   g++
*   libncurses-dev
*   flex
*   bison
*   libssl-dev
*   libelf-dev
*   lilo
*   texinfo
*   libisl-dev
*   m4
*   gcc-8-aarch64-linux-gnu

If you are on Ubuntu 18.04, you can run the following command:

```
sudo apt update && sudo apt install -y \   
  make \   
  gcc \   
  g++ \   
  libncurses-dev \   
  flex \   
  bison \   
  libssl-dev \   
  libelf-dev \   
  lilo \   
  texinfo \   
  libisl-dev \   
  m4 \   
  gcc-8-aarch64-linux-gnu
```

I'm assuming here that you also have the basic packages a common GNU/Linux distribution includes, like a way to download files, extract archives and edit files.

[Back to top](#top)

### Clean Environment

Because your shell may have environment variables that can cause issues, before running through this installation, always start with a clean shell. Invoking the code below will only have PATH and PS1 for environment variables, ignoring current environment variables and startup files.

`env -i PATH=$PATH PS1='Mezey@\w> ' /bin/bash --noprofile --norc`

[Back to top](#top)

### Global variables

Export these environment variables into your new shell.

```
export MEZEY_DIR=~/mezey   
export MEZEY_BUILD=x86_64-linux-gnu   
export MEZEY_TARGET=aarch64-linux-gnu   
export MEZEY_TMP=$MEZEY_DIR/tmp
```

[Back to top](#top)

### Partition a new hard drive

We need an extra disk so we can install Mezey GNU/Linux on it.  Since we need to download, extract and build the Linux Kernel, as well as setup Mezey GNU/Linux, this disk should be at least 30 GB.

> **Note** 
> 
> If you want to use a disk image instead of a SATA disk, skip to this section: [Using a disk image](#using-a-disk-image)

#### Using a SATA disk

Make sure your SATA disk is attached, and has been found. Run the command below to make sure this disk is listed, and what device name it was given.

`lsblk`

If it's a SATA device and plugged into port 1, it will probably be given the name _sdb_.

#### Using a disk image
> **Note** 
>
> Skip to [Creating a boot sector](#creating-a-boot-sector) if you are using a SATA disk

If you don't have an external disk attached to your computer, you can create a disk image.  First, create a 30 GB disk image in your home directory called mezey.img.

```
sudo dd if=/dev/zero of=~/mezey.img bs=1000M count=30
```

Now mount it as a loop device:

```
sudo losetup -fP ~/mezey.img
```

This will mount your disk to the first available loop device.  To determine what loop device is being used for your disk image, you can run the command `losetup`.  Your disk name will look something like `/dev/loop0` and the partition we created can be accessed at `/dev/loop0p1`.

#### Creating a boot sector

We will be using the tool fdisk to partition our new hard drive. You can technically use a different tool to partition your hard drive as long as you are able to create a Master Boot Record and a partition.

> **Warning**
> 
> The following commands will remove any data on the Mezey device so make sure you have the correct device name!   If you have a SATA disk, your device name may be /dev/sdb.  If you are using a loop device, your device name may be /dev/loop0.  May attention to the device name you give below.  Moving forward, I'm going to assume your Mezey disk is located at /dev/sdb.
  
> **Note**
> 
> This documentation does NOT discuss using EFI, although there is nothing preventing you from using this specification, I just won't be using it to setup Mezey GNU/Linux in this documentation. For more information on EFI, visit [https://en.wikipedia.org/wiki/Unified\_Extensible\_Firmware\_Interface](https://en.wikipedia.org/wiki/Unified_Extensible_Firmware_Interface).

To start fdisk, run the command:

`sudo fdisk /dev/sdb`

We need to create our boot sector (a classic generic Master Boot Record). This is not a partition, but rather a section of the disk right before any of the partitions. This sector will hold our bootable code (more on this later) which can be up to 466 bytes, a partition table and a boot signature. To create our generic Master Boot Record, type the character **o**.

Now let's create a single partition by typing the character **n**. This will then proceed to ask you if this is a _primary_ or _extended_ partition. Choose **p** for primary. Now select the default start sector which should be _2048_ and the default last sector which should be the end of your disk.

The last command we will type is **w**, which will write the partition table to our disk and exit.

> **Note**
> 
> Some users or GNU/Linux distributions prefer to create extra partitions. For example, Ubuntu has a partition for / and for /boot. Feel free to partition your Mezey disk anyway you want.

Before we can mount this partition, we need to create an ext4 file system on it. This can be done by using the command:

`sudo mkfs.ext4 /dev/sdb1`  
  
> **Warning**
> 
> Make sure you create a filesystem on our newly created partition or you will destroy our boot sector.  To be sure of the naming convention of the device's partition, we can use the command `lsblk`.  For example, if we're using a loop device, our output may look like: 

```
andrew@legion:~$ lsblk /dev/loop3
NAME      MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
loop3       7:3    0 29.3G  0 loop
└─loop3p1 259:0    0 29.3G  0 loop
```

In the output above, we want to create our filesystem here: `loop3p1`, so the command is `sudo mkfs.ext4 /dev/loop3p1`.

[Back to top](#top)

### Mount new partition

Let's create our root directory:

`sudo mkdir $MEZEY_DIR`

We will use this as our mount point for the new partition we just created above. To mount the partition, run the command:

`sudo mount /dev/sdb1 $MEZEY_DIR`  
  
> **Note**
> 
> For convenience, make sure the user you are logged in as has read and write permissions to the $MEZEY\_DIR directory. For example, if you are logged in as the default user, you can use the command below:
  
`sudo chown 1000:1000 $MEZEY_DIR`

[Back to top](#top)

### Mezey directory tree

Okay, now it's time to create some directories in our new partition.

```
mkdir $MEZEY_DIR/boot   
mkdir $MEZEY_DIR/etc/   
mkdir $MEZEY_DIR/tmp   
mkdir -p $MEZEY_DIR/usr/local/lib   
mkdir $MEZEY_DIR/usr/include   
mkdir $MEZEY_DIR/dev   
mkdir $MEZEY_DIR/cross   
mkdir -p $MEZEY_DIR/lib/modules/5.11-Mezey
```

[Back to top](#top)

### Get the Linux Kernel headers

Our next step is to get the Linux Kernel Headers. I will be using the 5.11 version. Feel free to use a newer or older version, but be warned, there could be differences between the versions that could cause the steps in this documentation to not work.

Run the following commands to download and extract the Linux Kernel source code:

```
cd $MEZEY_DIR/tmp   
wget https://cdn.kernel.org/pub/linux/kernel/v5.x/linux-5.11.tar.xz   
tar -xvf linux-5.11.tar.xz   
cd linux-5.11
```
  
> **Note**
> 
> I will NOT be discussing here how to verify the kernel signature. Review the documentation here for more information on how to do this: [https://www.kernel.org/category/signatures.html](https://www.kernel.org/category/signatures.html)

To generate the header files needed for the ARM64 architecture, run the command below:

`make headers_install ARCH=arm64 INSTALL_HDR_PATH=$MEZEY_DIR/usr`

After this command completes, you will have headers installed in:

*   $MEZEY\_DIR/usr/include/asm
*   $MEZEY\_DIR/usr/include/asm-generic
*   $MEZEY\_DIR/usr/include/drm
*   $MEZEY\_DIR/usr/include/linux
*   $MEZEY\_DIR/usr/include/misc
*   $MEZEY\_DIR/usr/include/mtd
*   $MEZEY\_DIR/usr/include/rdma
*   $MEZEY\_DIR/usr/include/scsi
*   $MEZEY\_DIR/usr/include/sound
*   $MEZEY\_DIR/usr/include/video
*   $MEZEY\_DIR/usr/include/xen

[Back to top](#top)

### GNU Binutils (Pass 1)

This is a collection of binary tools, most notably, a linker and an assembler. Before we compile this tool, there is a couple of important things to note. We need to tell binutils we want to cross-compile. We also have to compile this tool before GCC and the GNU C Library as the configure script in both of these tools actually runs tests against the assembler and linker, so when these tests are ran, they need to run against our new assembler and linker.

>**NOTE**
>
>  When dealing with cross-compiling, we have 3 properties we need to deal with; build, host and target.  Build is the architecture we are compiling the compiler on, host is the architecture the compiler will run on and target is the architecture the compiler will produce code for.  For binutils, we are building on x86_64-linux-gnu, which will run on x86_64-linux-gnu, but will produce code for aarch64-linux-gnu.

With that said, we can get this tool here:

```
cd $MEZEY_TMP   
wget https://ftp.gnu.org/gnu/binutils/binutils-2.36.tar.xz   
tar -xvf binutils-2.36.tar.xz
```

Now create a build directory for this tool:

`mkdir binutils-2.36-build && cd binutils-2.36-build`

and build/install it:

```
../binutils-2.36/configure \     
  --prefix="$MEZEY_DIR/cross" \     
  --with-sysroot=$MEZEY_TARGET \     
  --build=$MEZEY_BUILD \     
  --host=$MEZEY_BUILD \     
  --target=$MEZEY_TARGET      
make -j $(nproc)   
make install
```

Once done, you should have the following binaries installed:

*   $MEZEY\_DIR/usr/local/aarch64-linux-gnu/bin/ar
*   $MEZEY\_DIR/usr/local/aarch64-linux-gnu/bin/as
*   $MEZEY\_DIR/usr/local/aarch64-linux-gnu/bin/ld
*   $MEZEY\_DIR/usr/local/aarch64-linux-gnu/bin/ld.bfd
*   $MEZEY\_DIR/usr/local/aarch64-linux-gnu/bin/nm
*   $MEZEY\_DIR/usr/local/aarch64-linux-gnu/bin/objcopy
*   $MEZEY\_DIR/usr/local/aarch64-linux-gnu/bin/objdump
*   $MEZEY\_DIR/usr/local/aarch64-linux-gnu/bin/ranlib
*   $MEZEY\_DIR/usr/local/aarch64-linux-gnu/bin/readelf
*   $MEZEY\_DIR/usr/local/aarch64-linux-gnu/bin/strip
*   $MEZEY\_DIR/usr/local/bin/aarch64-linux-gnu-addr2line
*   $MEZEY\_DIR/usr/local/bin/aarch64-linux-gnu-ar
*   $MEZEY\_DIR/usr/local/bin/aarch64-linux-gnu-as
*   $MEZEY\_DIR/usr/local/bin/aarch64-linux-gnu-c++filt
*   $MEZEY\_DIR/usr/local/bin/aarch64-linux-gnu-elfedit
*   $MEZEY\_DIR/usr/local/bin/aarch64-linux-gnu-gprof
*   $MEZEY\_DIR/usr/local/bin/aarch64-linux-gnu-ld
*   $MEZEY\_DIR/usr/local/bin/aarch64-linux-gnu-ld.bfd
*   $MEZEY\_DIR/usr/local/bin/aarch64-linux-gnu-nm
*   $MEZEY\_DIR/usr/local/bin/aarch64-linux-gnu-objcopy
*   $MEZEY\_DIR/usr/local/bin/aarch64-linux-gnu-objdump
*   $MEZEY\_DIR/usr/local/bin/aarch64-linux-gnu-ranlib
*   $MEZEY\_DIR/usr/local/bin/aarch64-linux-gnu-readelf
*   $MEZEY\_DIR/usr/local/bin/aarch64-linux-gnu-size
*   $MEZEY\_DIR/usr/local/bin/aarch64-linux-gnu-strings
*   $MEZEY\_DIR/usr/local/bin/aarch64-linux-gnu-strip

And a bunch of files in:

*   $MEZEY\_DIR/usr/local/aarch64-linux-gnu/lib/ldscripts/
*   $MEZEY\_DIR/usr/local/lib/
*   $MEZEY\_DIR/usr/local/include/
*   $MEZEY\_DIR/usr/local/share/

[Back to top](#top)

### GCC, the GNU Compiler Collection Intro

The GNU compiler is a front-end for C and C++.

Goal: Create a compiler that runs on x86\_64 architecture (host) but will produce aarch64 (build) code.  

Before we proceed with compiling this tool, note we are building a degraded compiler. Why is this? The GCC compiler we are building will NOT have the Standard C Library (we haven't built it yet). However, this will be enough to compile the C Standard Library with. Once we build the C Standard Library, we will have a complete cross-compiler to build our tools (including a native compiler) that will allow us to enter a chroot environment (more on this later).

We also need to download some dependencies which we can download using the command below. By downloading these libraries into the GCC source directory, they will be statically linked when building GCC. If you skip this step, GCC will find these libraries elsewhere and dynamically link to them which will not work when trying to compile the C Library.

For more information on the GCC installation process, you can visit: [https://gcc.gnu.org/wiki/InstallingGCC](https://gcc.gnu.org/wiki/InstallingGCC)

[Back to top](#top)

#### Downloading the GCC, the GNU Compiler Collection

```
cd $MEZEY_TMP   
wget https://ftp.gnu.org/gnu/gcc/gcc-10.2.0/gcc-10.2.0.tar.gz   
tar -xvf gcc-10.2.0.tar.gz
```

#### Downloading GMP/MPFR/MPC

```
cd gcc-10.2.0   
./contrib/download_prerequisites
```

[Back to top](#top)

#### Building GCC, the GNU Compiler Collection

Create a build directory:

```
cd $MEZEY_TMP   
mkdir gcc-10.2.0-build   
cd gcc-10.2.0-build
```

Proceeding, we can now build GCC:
  
> **Note**
> 
> The configure option -with-newlib prevents the compiling of any code that requires the C Standard Library by telling the compiler that we will be using newlib instead, which is a lightweight C Standard Library for embedded systems. We won't be actually using newlib, but this is a way to compile GCC without the C Standard Library.
  
```
../gcc-10.2.0/configure \     
  --target=$MEZEY_TARGET \     
  --host=$MEZEY_BUILD \     
  --build=$MEZEY_BUILD \     
  --prefix=$MEZEY_DIR/cross \     
  --with-newlib \     
  --without-headers \     
  --enable-initfini-array \     
  --enable-languages=c,c++ \     
  --disable-nls \     
  --disable-shared \     
  --disable-multilib \     
  --disable-decimal-float \     
  --disable-threads \     
  --disable-libatomic \     
  --disable-libgomp \     
  --disable-libquadmath \     
  --disable-libssp \     
  --disable-libvtv \     
  --disable-libstdcxx      
make all -j $(nproc)   
make install
```  

Once done, you should have the following binaries installed:

*   $MEZEY\_DIR/bin/aarch64-linux-gnu-c++
*   $MEZEY\_DIR/bin/aarch64-linux-gnu-cpp
*   $MEZEY\_DIR/bin/aarch64-linux-gnu-g++
*   $MEZEY\_DIR/bin/aarch64-linux-gnu-gcc
*   $MEZEY\_DIR/bin/aarch64-linux-gnu-gcc-10.2.0
*   $MEZEY\_DIR/bin/aarch64-linux-gnu-gcc-ar
*   $MEZEY\_DIR/bin/aarch64-linux-gnu-gcc-nm
*   $MEZEY\_DIR/bin/aarch64-linux-gnu-gcc-ranlib
*   $MEZEY\_DIR/bin/aarch64-linux-gnu-gcov
*   $MEZEY\_DIR/bin/aarch64-linux-gnu-gcov-dump
*   $MEZEY\_DIR/bin/aarch64-linux-gnu-gcov-tool
*   $MEZEY\_DIR/bin/aarch64-linux-gnu-lto-dump

And a bunch of other files in:

*   $MEZEY\_DIR/lib/gcc/aarch64-linux-gnu/

> **Note**
> 
> As you can see in the files listed above, GCC is also providing the files ar and nm. This is not redudant. These are wrappers around the binaries provided by the binutils package. Since GCC supports plugins, this allows for dynamic loading a recognizer/analyser.

[Back to top](#top)

### GNU C Library

The C library provides API's we will use to talk to the Linux Kernel. We are going to build the C Library against our Linux Kernel headers we extracted in earlier steps, and build it using the cross-compiler we just built above. For more information on compile options, visit this page: [https://www.gnu.org/software/libc/manual/html\_node/Configuring-and-compiling.html](https://www.gnu.org/software/libc/manual/html_node/Configuring-and-compiling.html).

Run the commands below to download and extract the C library:

```
cd $MEZEY_DIR/tmp/   
wget http://mirrors.kernel.org/gnu/libc/glibc-2.33.tar.xz   
tar -xvf glibc-2.33.tar.xz   
```

To compile the C library, we need to create a build directory:

`mkdir glibc-2.33-build/ && cd glibc-2.33-build/`

Now let's build the C Library:

```
CC=$MEZEY_DIR/cross/bin/aarch64-linux-gnu-gcc ../glibc-2.33/configure \     
  --with-headers=$MEZEY_DIR/usr/include \     
  --enable-kernel=5.11 \     
  --build=$MEZEY_BUILD \     
  --host=$MEZEY_TARGET \     
  --disable-sanity-checks\     
  --disable-werror     
make -j $(nproc)   
make install DESTDIR=$MEZEY_DIR
```

[Back to top](#top)

### Setup GNU BASH

Everyone needs a shell... Well, almost everyone. For Mezey GNU/Linux, we are going to install GNU bash. This shell can be downloaded here:

`wget ftp://ftp.gnu.org/gnu/bash/bash-5.1.tar.gz`

[Back to top](#top)

### Setup GNU Coreutils

In order to use things like ls, cd, etc., we need a bunch of utilities. Luckily for us, there is GNU Coreutils which has a lot of utilities we can use for our Mezey GNU/Linux installation.

[Back to top](#top)

### Compile the Linux Kernel

Our next step is to compile the Linux Kernel. I will be using the 5.11 version. Feel free to use a newer or older version, but be warned, there could be differences between the versions that could cause the steps in this documentation to not work.

The Linux Kernel has a lot of configurations. Too many to list and discuss here. If you know about the different Linux Kernel configurations and want to make some adjustments, feel free.

We however need to make some adjustments. This is also a good time to discuss the Linux Kernel boot process. In order for the Linux Kernel to read our Mezey disk, it needs to load some drivers so it can understand how to read the disk. However, these drivers are located on the disk it first needs to understand. It's a chicken and egg problem. To solve this problem, we have two solutions. The first solution which I think is the easiest, is to compile those drivers directly into the Linux Kernel. If you do NOT like the idea of compiling these drivers into the Linux Kernel, you can go with the second option, which is to create a RAM disk that will hold these early boot drivers. The boot loader will mount this RAM disk during the boot process, and from there, the Linux Kernel will have access to this temporary file system with the necessary drivers to load the Mezey disk and the permanent file system. I will not be creating a RAM disk. I will build the Linux Kernel with these drivers.

To load the configuration menu, run the command:

`make menuconfig`

Go to _Device Drivers_ -> _Serial ATA and Parallel ATA drivers (libra)_ and for the option _AHCI SATA support_, set the value to _built-in_. This means this driver will be compiled into the Linux Kernel.

Exit these sub menus and go to _File systems_. Make sure the _ext2_ and _ext3_ file systems are set to _built-in_.

Exit this sub menu and got to _General Setup_ -> _Compile-time checks and compiler options_. Make sure _Install uapi headers to usr/include_ is set to _built-in_.

Save your configurations and exit.

Now that we are happy with our configurations, we can start to build the Linux Kernel.

`make -j $(nproc)`  

Once the compilation process is done, we can compress the Linux Kernel:

`make bzImage`

[Back to top](#top)

### Install Linux Kernel and its drivers

The following commands will install the Linux kernel and the kernel modules onto the Mezey disk.

```
export INSTALL_PATH=$MEZEY_DIR/boot/   
export INSTALL_MOD_PATH=$MEZEY_DIR/lib/modules/5.11-Mezey   
make install   
make modules_install
```

[Back to top](#top)

### Create a boot loader

You can technically use any boot loader you like, but I will be using LILO. It's simple, and lightweight (but also deprecated).

First thing first, create the file `/Mezey/etc/lilo.conf` and add the following contents:

```
disk=/dev/sdb bios=0x80   
boot=/dev/sdb   
map=/boot/map   
install=/boot/boot.b   
image=/boot/vmlinuz-5.11.0     
label=Mezey-linux     
root=/dev/sda1
```

Now we need to create the boot loader, and some other configuration files by running the commands:

```
sudo umount /boot   
sudo mount -bind $MEZEY_DIR/boot /boot   
sudo lilo -C $MEZEY_DIR/etc/lilo.conf   
```

This command will add the LILO boot loader to the first sector of our Mezey disk, and also create a couple of configuration files in our boot directory.

> **Warning**
> 
> Be carful, and make sure disk and boot point to your Mezey disk device name, or you may overwrite your current boot loader!

[Back to top](#top)

### Verify Mezey GNU/Linux boot directory

So far, our boot directory should look like below:

`./boot.0810   ./config-5.11.0   ./map   ./System.map-5.11.0   ./vmlinuz-5.11.0`

[Back to top](#top)

### Simple test init

To confirm we have a working system, we are going to make a simple init system that will just print the text _Welcome to Mezey GNU/Linux!_ when the Linux Kernel boots. This will be simple, as we only need a few lines of code:

```c
int main(int argc, char** args) {     
    printf("Hello from Mezey GNU/Linux!");     
    return 0;   
}
```

Save this code to a file called _$MEZEY\_DIR/init.c_. To compile this code, we want to make sure we use the new C Library we just compiled:

```bash
gcc -Xlinker -rpath=$MEZEY_DIR/usr/local/lib \     
    -Xlinker -I$MEZEY_DIR/usr/local/lib/ld-2.33.so init.c
```

And now copy this file to our _/sbin_ directory:

`cp a.out $MEZEY_DIR/sbin/init`

It is at this point I like to verify we have a bootable disk running the LILO boot loader and the Linux Kernel. To verify, we need to shutdown the machine and attach our Mezey disk to SATA port 0 of the computer.

Doing so, you should see the message _Hello from Mezey GNU/Linux!_ If you see this, congrats! You have a working Linux Kernel with the C Standard Library. If you see an error message that complains about _Unable to mount root fs on unknown-block..._, this is most likely caused by the Linux Kernel not understanding the SCSI disk. Make sure you have complied the Linux Kernel correctly with the necessary drivers built-in.

[Back to top](#top)

### Setup our real init system

I will be going with a SysV style init system. Feel free to use Systemd, but I will not be discussing it here.

[Back to top](#top)

### Using Chroot

Before proceeding to the next step, we need to use a tool called chroot. This will make things easier moving forward as we don't need to do any special linking to our Mezey directory. We will trick the system into thinking that the Mezey directory is actually our root directory.

[Back to top](#top)

### User Setup

[Back to top](#top)

### Setup Networking

[Back to top](#top)

### Create an ISO Image

[Back to top](#top)

### Setting up the mezpkg tool

[Back to top](#top)

### Anatomy of a mezpkg file

[Back to top](#top)

### Installing mezpkg manager

[Back to top](#top)

### How to use mezpkg manager

[Back to top](#top)

### Notes

[Back to top](#top)

### Known issues

[Back to top](#top)

### Trouble shooting

[Back to top](#top)
