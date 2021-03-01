# So you want to install Gentoo?

![Saint iGNUcius](https://i.imgur.com/V67I2v7.png)

My aim is to make your installation as simple as possible. This will start out as a series of instructions, but maybe some day I'll create some scripts to help you better.

This information will be mostly taken from the excellent youtube video by [Mental Outlaw](https://www.youtube.com/watch?v=6yxJoMa05ZM&feature=emb_title), but more generalised than the video he made for his own PC. When I say generalised, I mean that I'll do this for my own PC, but I'll explain each step along the way.

This guide will be for x86_64 PCs (which Gentoo calls amd64, and is almost certainly what your computer's architecture is). More details can be found in the [Gentoo AMD64 Handbook](https://wiki.gentoo.org/wiki/Handbook:AMD64) if you desire.

## Getting Started

If you're not already running Linux, you'll need to find a live image, which is where you'll be performing the installation from. This image needs burned onto a USB flash drive or a CD (I recommend the flash drive). You can do this directly from Windows, or you can use a tool like Balena Etcher if you prefer.

If you are installing Gentoo on a system that's already running Linux, a lot of these steps will be the same. The main difference is you'll probably need to run the commands as a root user, by starting with the command *sudo su*. However, this guide will assume you are installing from a live image instead, which probably won't require this.

You can download the latest live image [here](https://www.gentoo.org/downloads/) and burn it onto your USB. You'll then plug this into the PC you're installing to, reboot the PC, and boot from the USB. Your computer will probably not do this by default, in which case you'll need to open the BIOS menu, and change which medium to boot from.

## The First Steps

Once you've booted from the USB, you'll be brought to a terminal on a mostly-black screen. Before we go any further, try pinging www.gentoo.org by running:

```
ping -c 10 www.gentoo.org
```

If the pings do not work, then you have a problem. Good luck fixing that, because I'm not going to. If the pings works, you can continue reading.

The next step is finding the target medium for installation, such as an internal hard drive (which this guide will assume). To find your hard drive, run:
```
fdisk -l

Disk /dev/sdb: 1.84 TiB ...
# (You'll find lines like this in the output)
```

This will show you all the disks on your system, as well as how it partitioned. Find the disk that corresponds to your hard drive (e.g. my hard drive is */dev/sdb*). **Make sure you don't mix these up, or you'll format the wrong hard drive** (which I'm ashamed to admit I've done in the past).

## Preparing the Hard Drive

If your hard drive is not already partitioned, this step will show you how. To make sure the hard drive is completely unpartitioned before installation, run this command, **replacing /dev/sdb with whatever your hard drive is**:
```
wipefs -a /dev/sdb
```

Once that's done, we'll need to use the *parted* program to format the hard drive, which just means we're going to split it into logically-divided parts. To begin, run this, **replacing /dev/sdb with whatever your hard drive is**:
```
parted -a optimal /dev/sdb
```

From here, you'll split your hard drive into partitions. There's a million ways you could do this, and this guide will use a different scheme than both the Gentoo wiki and Mental Outlaw use; I will be creating a separate */home* partition in addition to the partitions these guides use, because I want to keep all my personal files completely separate from system files, among many other reasons that I won't discuss in this guide. Some people take this further by having directories like */var* (for logs, emails, etc.) and */tmp* (for temporary files) be their own partitions, but I won't in this guide.

You must also check if your motherboard uses BIOS or UEFI, because this affects how you create these partitions and what file systems you'll be using. My PC (as well basically all modern PCs) use UEFI, so that's what I will be using in this guide.

This is how I will be partitioning my hard drive, and the next steps will show you how to set this up:
* boot
* grub
* swap
* root
* home


Change the unit we use when making partitions to megabytes.
```
unit mib
```
Create a gpt label, which is an indentifier for this hard drive.
```
mklabel gpt
```
Create the partition used by GRUB (the bootloader) when booting the system. It is 2MB big, starting from the first MB of the drive and ending on the third.
```
mkpart primary 1 3
```
Name this new partition (which is partition 1) 'grub'
```
name 1 grub
```
Set the bios_grub flag, which lets GRUB know to boot from here.
```
set 1 bios_grub on
```
Create a boot partition, which is used by the Linux kernel, the root file system, GRUB configuration, etc. This is not needed on newer systems, but there are upsides to having this partition (such as using RAID or hard drive encryption), so I will make it anyway. Everyone will give you different values for how big /boot should be, but I will follow the Gentoo wiki's recommendation of 128MB.
```
mkpart primary 3 131
```
Name this partition 'boot'
```
name 2 boot
```
Create a swap partition. This is used by the kernel to copy files from RAM to disk, to free up RAM. It's also used when the system runs out of RAM entirely. It's not strictly necessary, but if you don't have a swap partition and you run out of RAM, your system will crash. Ultimately, the more RAM you have, the smaller a swap file you'll probably need. 

However, if you want to be able to hibernate your system, /swap will need to be at the very least as big as RAM, so I will go for this, plus an extra gigabyte to be safe. If we let R be the amount of RAM you have in MB, then we will make our partition span from 131 to (131 + 1024 * (R + 1)). For example, I have 12GB of RAM, so my partition will end at (131 + 1024 * (12 + 1)), which for me is 13443.
```
mkpart primary 131 13443
```
Name this partition 'swap'
```
name 3 swap
```
Create the root partition, which is where everything required for a working Linux system resides, as well as all files that aren't on a separate partition, such as third-party binaries. If you don't want a /home partition, this occupies all the remaining disk space, but we want a separate home partition. I'm going to install a lot of stuff on my PC, so I want plenty of space in my /root partition, so I'm choosing 64GB arbitrarily. Most people say to use 20GB to 30GB, but I have plenty of disk space, so I'm going with 64GB.
**Replace 13443 with the value you calculated in the previous step. The end value will be that value plus 6553**
```
mkpart primary 13443 78979
```
Name this partition 'root'
```
name 4 root
```
Now, we can fill the rest of the hard drive with the /home partition, which will contain all your user's files. To use all the remaining space, we use *-1* as the last argument to mkpart.
```
mkpart primary 78979 -1
```
Name it '/home'
```
name 5 home
```

Now that you've set up all your partitions, run 'print' and display the output. After the commands I ran, my output looks like this:
```
print

Model: ATA WDC WD20EZRX-OOD (scsi)
Disk /dev/sdb: 1907729MiB
Sector size (logical/physical): 512B/4096B
Partition Table: gpt
Disk Flags:

Number  Start     End         Size       File system     Name  Flags
1       1.00MiB   3.00MiB     2.00MiB    fat32           grub  bios_grub
2       3.00MiB   131MiB      128MiB     fat32           boot
3       131MiB    13443MiB    13312MiB   linux-swap(v1)  swap
4       13443MiB  78979MiB    65536MiB                   root
5       78979MiB  1907728MiB  1828749MiB                 home
```

Yours should look very similar to this. Check the size of each partition, the name, and the labels, to make sure everything is correct. Then, quit out of parted using 'quit':
```
quit
```

As an additional sanity check, you can run *lsblk* to make sure you formatted the correct hard drive.

## Creating the File Systems

All your partitions will need the correct file systems on them. I will avoid the details of each kind of file system; basically you want to use FAT32 for boot, and ext4 for root and home.
```
mkfs.fat -F 32 /dev/sdb2
mkfs.ext4 /dev/sdb4
mkfs.ext4 /dev/sdb5
```

The swap partition requires different commands.
```
mkswap /dev/sdb3
swapon /dev/sdb3
```

## Mounting the file systems

To actually use these file systems and install Gentoo, you need to mount the root partition, which means assigning it to a path somewhere on your PC. We will also mount the home partition.
```
mount /dev/sdb4 /mnt/gentoo
mkdir /mnt/gentoo/home
mount /dev/sdb5 /mnt/gentoo/home
```

Before the next step, make sure your date and time are set correctly. You can check this by running:
```
date
```
If the date and time are not correct, you'll need to set them either:
* Manually, by using the *date* command
* Automatically, by connecting to an ntp server with *ntpd*


## Downloading Gentoo

Gentoo distributes itself in what it calls a *stage 3 tarball*. This contains the libraries and tools you need to bootstrap the system. It also contains the basis for a particular kind of system, so you have multiple options:
1. Stage 3 (openrc) - This is what most people will choose.
2. Stage 3 (systemd) - Uses systemd instead of openrc for managing services. This guide assumes you're using openrc.
3. Stage 3 (no multilib | openrc) - Same as 1, but don't ever use 32-bit libraries.
4. Stage 3 (uclibc | openrc) - Uses the uclibc standard C library.
5. hardened Stage 3 (openrc) - Uses an alternative project called *Gentoo Hardened*, which is like Gentoo but with additional security features. **This is what I'll be using in this guide.**
6. Hardened Stage 3 (no multilib | openrc)
7. Hardened Stage 3 (uclibc | openrc)
8. Stage 3 (x32 | openrc)

For this guide, we will be using option 5, since I'd like some of those extra security features. Since you don't have a web browser when installing from a live CD, you'll need to go to the [download](https://www.gentoo.org/downloads/#other-arches) page, find the link to the openrc tarball, then paste it into wget. It will probably be the first option listed under *stage archives*. **Replace the argument to wget with the download link for the latest tarball.**
```
cd /mnt/gentoo
wget https://bouncer.gentoo.org/fetch/root/all/releases/amd64/autobuilds/20210224T214503Z/stage3-amd64-20210224T214503Z.tar.xz
```

The next step is to unpack this tarball.
```
tar xpvf stage3-amd64-20210224T214503Z.tar.xz --xattrs-include='*.*' --numeric-owner
rm stage3-amd64-20210224T214503Z.tar.xz
```

## make.conf

Before we proceed with the installation, copy the provided *make.conf* file to the configuration in */mnt/gentoo/etc/portage/make.conf*.

```
wget https://github.com/Razorfang/MyGentoo/raw/main/make.conf
mv make.conf /mnt/gentoo/etc/portage/make.conf
```

Gentoo uses a package manager and distribution system called *portage*, which allows you to manage packages, install and build packages differently depending on different flags, and change some pre-configured Gentoo settings.

A lot of your customisation will be done from a file called *make.conf*, which is stored in */etc/portage/make.conf* on most Gentoo systems. How you configure this file is entirely system-dependent, but this repository contains my own *make.conf*, commented to help you make your own decisions.

For example, there is a variable called *USE* which controls which features packages are built with by default, *VIDEO_CARDS* which controls supported GPUs, and *GRUB_PLATFORMS* to install the correct version of GRUB during this installation.

You can read the man page for make.conf to see every supported flag, which will be useful for tweaking your own installation.


## Preparing the Environment

It's almost time to start work inside the new Gentoo environment. Before entering, you'll need to copy some networking settings.
```
cp --dereference /etc/resolv.conf /mnt/gentoo/etc/
```

Next, you need to copy the list of repositories that you'll get everything from.
```
mkdir --partents /mnt/gentoo/etc/portage/repos.conf
cp /usr/share/portage/config/repos.conf /mnt/gentoo/etc/portage/repos.conf/gentoo.conf
```

The next step is mounting all the file systems the kernel will need; /proc/, /sys/, and /dev/.
```
mount --types proc /proc /mnt/gentoo/proc
mount --rbind /sys /mnt/gentoo/sys
mount --make-rslave /mnt/gentoo/sys
mount --rbind /dev /mnt/gentoo/dev
mount --make-rslave /mnt/gentoo/dev
```

Now you need to enter the new environment. After this point, all actions will be performed from inside the new Gentoo environment.
```
chroot /mnt/gentoo /bin/bash
source /etc/profile
export PS1="(chroot) ${PS1}"
```

**------------------------------------------------------------------------------------------------------**
**If you significantly mess up any stage beyond this point, you should be able to start again from here.**
**------------------------------------------------------------------------------------------------------**

Anyway, you'll need to mount the boot directory to perform the next steps:
```
mount /dev/sdb2 /boot
```

## Emerge

The next step is to install a snapshot of the gentoo ebuild repository. This is used by Portage to help build your system. To do this:
```
emerge-webrsync
```

You can now use the *emerge* command-line tool to install packages. But first, we need to choose our *profile*. Think of a profile as a building block for a Gentoo system. It sets how packages are built by default, default compiler options, allowed package versions, etc. It is possible, but non-trivial, to change profiles later, so keep that in mind.

To list all the profiles, do this command. Your output may differ:
```
eselect profile list

Available profile symlink targets:
  [1]  default/linux/amd64/17.1 (stable) *
  [2]  default/linux/amd64/17.1/selinux (stable)
  [3]  default/linux/amd64/17.1/hardened (stable)
...
```

The star means this is your currently selected profile. I will be using the **hardened** profile. To select it, I will run:
```
eselect profile set 3
# Or whatever number corresponds to basic hardened profile.
```

After selecting a profile, you must update the system's *@world set*, which is just all the packages a profile uses. You should also use this if you ever make any changes to the USE flag in your make.conf.

Running the following command will take a long time, so it's the perfect time to have a break. If you want to run it and see what packages would be installed, add the *-pv* flags at the end. This way, you can double-check which USE flags these packages will install with. To actually install the packages, remove the *-pv*.
```
emerge --ask --verbose --update --deep --newuse [-pv] @world
```

Optionally, you can replace the default text editor (Nano) with something else. I prefer using Vim, so I will replace Nano with Vim in this step:
```
emerge -q app-editors/vim
```

## Setting your Timezone and Locale

The next step is updating your timezone, to make sure your PC displays the correct time after installation. If you live in a different time zone than I do, please look up what yours is by typing *ls /usr/share/zoneinfo*
```
echo "Pacific/Auckland" > /etc/timezone
emerge --config sys-libs/timezone-data
```

Now we're going to set our locale, which defines how dates are displayed, what alphabet you use, etc. This guide contains a New Zealand locale.gen file, but you can edit it for your own locale
```
cd /tmp
wget https://github.com/Razorfang/MyGentoo/raw/main/locale.gen
cp /tmp/locale.gen /etc/locale.gen
rm /tmp/locale.gen
locale-gen
eselect locale list
# Find your locale in the list. In my case, I want en_NZ.utf8, which has index 6
eselect locale set 6
env-update
source /etc/profile && export PS1="(chroot) ${PS1}"
```

## Kernel configuration

Gentoo encourages users to configure the Linux kernel to their own liking. Don't worry, it sounds harder than it is. You could use *genkernel* to automatically configure it, but this guide will opt for manual configuration.

First, install the Linux kernel source code.
```
emerge --ask sys-kernel/gentoo-sources
```

Next, install pciutils, which is required if you are using PCI devices (such as graphics cards)
```
emerge -q sys-apps/pciutils
```

Now, before we go any further, you must know what your PC is like. You need to know what you want, and don't want, to support. Turning off random settings will cause a ton of trouble, so only turn off settings you KNOW you don't want (e.g. AMD drivers since I'm using an Nvidia graphics card). I will post my kernel configuration in this guide, but **it is tailored for my PC, and may or may not work for you**. Please check my configuration against what you need. If you are not sure, just skip the all custom configuration, since you can do that later.

You can find more information about kernel configuration [here](https://wiki.gentoo.org/wiki/Kernel/Gentoo_Kernel_Configuration_Guide). Strictly speaking, kernel configuration is not essential, but it is recommended.

To begin kernel configuration, you need to run this:
```
cd /usr/src/linux
make menuconfig
```

This will bring you into a large menu, full of options under more options under even more options. Some examples of my configuration are:
* Disabled wifi support
* Disabled AMD microcode support
* Enabled kernel virtualisation (KVM)

Make your own changes and save them. Afterwards, we build the kernel.
```
make
make modules_install
make install
```

## Making an initramfs

To make it easier for the kernel to boot your system, we will create what's called an *initramfs*. This is basically an archive of information used by the kernel before your system's init system (like OpenRC) starts.

```
# If you copied my config, specify the full path to that file as the argument to --kernel-config instead
# --lvm means enable LVM support and --mdadm enables software RAID
emerge --ask sys-kernel/genkernel
genkernel --lvm --mdadm --install --kernel-config=/usr/src/linux/.config initramfs

# This step installs additional firmware for devices like network interfaces.
emerge --ask sys-kernel/linux-firmware
```

## Creating /etc/fstab

All partitions of your system must be listed inside the file */etc/fstab*. To configure this file, you will need to find the *UUID* of each partition, which is a unique identifier for that partition. To find the UUIDs, run this command, **replacing /dev/sdb with your hard drive's name**:
```
blkid | grep /dev/sdb
```
**This repository contains my /etc/fstab file. Replace the UUID of each partition with your own UUIDs, then copy it to /etc/fstab.**

## Networking Configuration

TO BE CONTINUED
