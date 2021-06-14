# So you want to install Gentoo?

![Saint iGNUcius](https://i.imgur.com/V67I2v7.png)

My aim is to make the installation of a Gentoo system relatively painless. This will start out as a series of instructions, but maybe some day I'll create some scripts to help you better. Probably not though.

This information will be mostly taken from the excellent youtube video by [Mental Outlaw](https://www.youtube.com/watch?v=6yxJoMa05ZM&feature=emb_title), but more generalised than the video he made for his own PC. When I say generalised, I mean that I'll do this for my own PC, but I'll explain each step along the way.

This guide will be for x86_64 PCs (which Gentoo calls amd64, and is almost certainly what your computer's architecture is). More details can be found in the [Gentoo AMD64 Handbook](https://wiki.gentoo.org/wiki/Handbook:AMD64) if you desire. 

## Getting Started

If you're not already running Linux, you'll need to find a live image, which is where you'll be performing the installation from. This image needs burned onto a USB flash drive or a CD (I recommend the flash drive). You can do this directly from Windows 10, or you can use a tool like Balena Etcher if you prefer.

If you are installing Gentoo on a system that's already running Linux, and it's being installed on a new hard drive, you can perform the same installation process from your host machine. Just know that you'll have to run the commands as root by using **sudo**.

You can download the latest live image [here](https://www.gentoo.org/downloads/) (it's under **Boot media**) and burn it onto your USB. You'll then plug this into the PC you're installing to, reboot the PC, and boot from the USB. Your computer will probably not do this by default, in which case you'll need to open the BIOS menu, and change which medium to boot from.

## The First Steps

Once you've booted from the USB, you'll be brought to a terminal on a mostly-black screen. Before we go any further, try pinging www.gentoo.org by running the following command. If this is successful, it means we will be able to download the files needed for our installation.

```
ping -c 10 www.gentoo.org
```

The next step is finding the target medium for installation, such as an internal hard drive (which this guide will assume). To find your hard drive, run:
```
fdisk -l

Disk /dev/sdb: 1.84 TiB ...
# (You'll find lots of lines like this in the output)
```

This will show you all the disks on your system, as well as how it partitioned. Find the disks that correspond to your hard drives. **Make sure you don't mix these up, or you'll format the wrong hard drive**.

## Preparing the Media

Hard drives are split into *partitions*, which are logical divisions between parts of the drive. These partitions are used by the system for different purposes. To properly perform this step, you will need to know how each hard drive will be used.

I will be using one SSD for the OS, and two HDDs for storage. Here is how I plan to organise my system. This guide will be setting exactly this up, but your steps can be modified depending.

You must also check if your motherboard uses BIOS or UEFI, because this affects how you create these partitions and what file systems you'll be using. My PC (as well basically all modern PCs) use UEFI, so please check the Gentoo wiki for what to do if you're using BIOS.

* 128GiB SSD
  * 2MiB /grub partition, used by the *GRUB* program to bootload the system
  * 128MiB /boot partition, used by Linux during booting
  * 24GiB /swap partition, used to hibernate the system, and is used when the system runs out of RAM.
  * rest is the /root partition, which is the core of the system.
* 2TiB HDD consisting of one large /home partiton, which is occupied by user files
* 3TiB HDD consisting of one large partition called /bulk, which I will use for storing large files like games and movies


To make sure the hard drive is completely unpartitioned before installation, run this command, **replacing /dev/sdX with whatever your hard drives are.** You can find these paths with the *lsblk* command.

```
# Wipe the SSD
wipefs -a /dev/sda
# Wipe the 2TiB drive
wipefs -a /dev/sdb
# Wipe the 3TiB drive
wipefs -a /dev/sdc
```

Once that's done, we'll need to use the *parted* program to format the hard drives.

```
# Partition the SSD
parted -a optimal /dev/sda
# Change working units to MiB
unit mib
$ Create identifier for the drive
mklabel gpt
# Create 2MiB grub partition
mkpart primary 1 3
# Name this partition 'grub'
name 1 grub
# Let GRUB know to boot from here
set 1 bios_grub on

# Create a 2MiB /boot partition
mkpart primary 3 131
# Name this partition 'boot'
name 2 boot

# Create a 24GB /swap partition
# Substitute (GB of RAM) with the RAM you have
mkpart primary 131 [131 + 1024 * 2 * (GB of RAM)]
# Name this partition 'swap'
name 3 swap

# Fill the rest of the SSD with the /root partition
mkpart primary [131 + 1024 * 2 * (GB of RAM)] -1
# Name this partition 'root'
name 4 root

# Verify layout with the print command.
# Check everything looks like it's partitioned correctly.
print
quit

# Partition the 2TiB HDD
parted -a optimal /dev/sdb
unit mib
mklabel gpt
mkpart primary 1 -1
name 1 /home
print
quit

# Partition the 3TiB HDD
parted -a optimal /dev/sdc
unit mib
mklabel gpt
mkpart primary 1 -1
name 1 /bulk
print
quit
```

## Creating the File Systems

All your partitions will need the correct file systems on them. Basically you want to use FAT32 for boot, and ext4 for everything except the swap partition.
I obtained the names of */dev/sdX* using the *lsblk* command.
```
# Create a FAT32 file system for the /boot partition on the SSD
mkfs.fat -F 32 /dev/sda2
# Create an ext4 file system for the /root partition on the SSD
mkfs.ext4 /dev/sda4
# Create an ext4 file system for the /home partition on the 2TiB HDD
mkfs.ext4 /dev/sdb1
# Create an ext4 file system for the /bulk partition on the 3TiB HDDs
mkfs.ext4 /dev/sdc1
```

The swap partition requires different commands.
```
# Set /swap as a swap partition on the SSD
mkswap /dev/sda3
swapon /dev/sda3
```

## Mounting the file systems

To actually use these file systems and install Gentoo, you need to mount the root partition, which means assigning it to a path somewhere on your PC. We will also mount the home partition.
```
mount /dev/sda4 /mnt/gentoo
mkdir /mnt/gentoo/home
mount /dev/sdb1 /mnt/gentoo/home
```

Before the next step, make sure your date and time are set correctly. You can check this by running:
```
date
```
If the date and time are not correct, you'll need to set them either:
* Manually, by using the *date* command
* Automatically, by connecting to an ntp server with *ntpd*


## Downloading Gentoo

Gentoo distributes itself in what it calls a *stage 3 tarball*. This contains the libraries and tools you need to bootstrap the system. It also contains the basis for a particular kind of system, so you have multiple options, such as whether to use openrc or systemd, whether to use a hardened kernel or not, and whether to deny 32-bit libraries.

For this guide, I will be choosing the basic option. Since you don't have a web browser when installing from a live CD, you'll need to go to the [download](https://www.gentoo.org/downloads/#other-arches) page, find the link to the openrc tarball, then paste it into wget. It will probably be the first option listed under *stage archives*. **Replace the link next to wget with the download link for the latest tarball.**
```
cd /mnt/gentoo
wget https://bouncer.gentoo.org/fetch/root/all/releases/amd64/autobuilds/20210611T113421Z/stage3-amd64-20210611T113421Z.tar.xz
```

The next step is to unpack this tarball.
```
tar xpvf stage3-amd64-20210224T214503Z.tar.xz --xattrs-include='*.*' --numeric-owner
rm stage3-amd64-20210224T214503Z.tar.xz
```

## make.conf

Gentoo uses a package manager and distribution system called *portage*, which allows you to manage packages, install and build packages differently depending on different flags, and change some pre-configured Gentoo settings.

A lot of your customisation will be done from a file called *make.conf*, which is stored in */etc/portage/make.conf* on most Gentoo systems. How you configure this file is entirely system-dependent, depending on any graphics cards you use, what software licenses you want to use, among other things.

For example, there is a variable called *USE* which controls which features packages are built with by default, *VIDEO_CARDS* which controls supported GPUs, and *GRUB_PLATFORMS* to install the correct version of GRUB during this installation.

You can read the man page for make.conf to see every supported flag, which will be useful for tweaking your own installation.

Here is what my make.conf looks like:
```
# This file is used by Portage to customise your packages.
# Check the manual page for make.conf or the gentoo wiki for more flags.

# These flags are used by your PC's C, C++, and Fortran compilers.
# In this file, all these compilers will use the same flags.
# --march=native will compile all packages for your PC's architecture, 
#    ensuring nothing is ever cross-compiled.
# -O2 means our compiler will try and optimise code during compilation.
#    It's not recommended to go to -O3 for all packages, so we'll go
#    with the -O2.
# -pipe dictates how different stages of compilation communicate.
#    -pipe means use pipes instead of temporary files.
COMMON_FLAGS="-march=native -O2 -pipe"
CFLAGS="${COMMON_FLAGS}"
CXXFLAGS="${COMMON_FLAGS}"
FCFLAGS="${COMMON_FLAGS}"
FFLAGS="${COMMON_FLAGS}"

# Defines additional software licenses for us to use.
# I am trying to use more FOSS software, so I only want to packages that Gentoo considers free software.
# The '-*' means 'only use the following licenses'
ACCEPT_LICENSE="-* @FREE"

# Changes Portage's default behaviour
# * fail-clean: Clean temporary directories after a build failure
FEATURES="fail-clean"

# Stores where ebuild logs are kept.
PORTAGE_LOGDIR="/var/log/ebuild-logs"

# MAKEOPTS determines how many parallel compilations should occur when
# installing a package. It's usually not worth making this higher than the
# number of cores your CPU has (in my case, four).
# -j is the number of cores.
# -l is the maximum allowed load average, which should be about j*0.9
MAKEOPTS="-j4 -l3.6"
# Similar story with this one
EMERGE_DEFAULT_OPTS="${EMERGE_DEFAULT_OPTS} --jobs=4 --load-average=3.6"

# This is used to tell Portage what graphics card you are using (if any).
# I am using an NVIDIA GTX970, so my choices are:
# * 'nouveau' for the open-source drivers
# * 'nvidia' for the proprietary drivers
VIDEO_CARDS="nouveau"

# This is used to tell Portage what we use to control our PC.
# libinput is for mice and keyboards.
INPUT_DEVICES="libinput"

# Tells the system which version of GRUB to install.
GRUB_PLATFORMS="efi-64"

# Sets the language of the build output to English
LC_MESSAGES=C

# Portage uses the USE variable to determine the default way to build
# a package. You can add/remove flags using the USE variable in this file.
# NOTE: This does not overwrite the value of USE that Portage uses, it only
# appends to it! It also applies to ALL packages you build, so only
# put a flag here if you want it to apply system-wide!
#  alsa:       Add support for the advanced linux sound architecture.
#  elogind:    Build packages that normally require logind from systemd.
#  examples:   Download code examples where applicable.
#  ipv6 :      Add support for ipv6.
#  multilib:   Allow compilation of 32-bit binaries.
#  nouveau:    Build all packages to make use of open-source nouveau drivers.
#  nvidia:     Enable nvidia-specific features.
#  opengl:     Add support for openGL graphics.
#  pulseaudio: Add support for PulseAudio.
#  qt4:        Add support for the Qt4 framework.
#  qt5:        Add support for the Qt5 framework.
#  udev:       Enable virtual/udev integration.
#  X:          Add support for X11.
#  -emacs:     Don't add support for emacs, since I use vim.
#  -gnome:     Don't add GNOME support.
#  -systemd:   Don't use systemd-specific libraries.
#  -wifi:      Don't enable wifi functionality.
USE="alsa elogind examples ipv6 multilib nouveau nvidia opengl pulseaudio qt4 qt5 udev X -emacs -gnome -systemd -wifi"

```


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

Optional: Change your PC's name by editing */etc/conf.d/hostname*. This isn't an IP address; it's a nice name like "GentooBox" or "MyPC".

Now we're going to configure a DHCP client. Please note, this guide will NOT cover configuring wi-fi, since I won't be using it on my PC.

First, download this:
```
emerge --ask --noreplace net-misc/netifrc
```
Next, we're going to specify which interface we're using. In my case, I'm using a wired connection over the interface *enp3s0*. To find which interface you should use, run *ifconfig* or *ip addr* and find the interface with your PC's IP address on it. Open the file */etc/conf.d/net*, and write the following, replacing <interface> with your interface:
```
config_<interface>="dhcp"
```

Next we will make sure the system runs DHCP at boot:
```
emerge net-misc/dhcpcd
cd /etc/init.d
ln -s net.lo net.<interface>
rc-update add net.enp3s0 default`
```

## Other Steps

While we're at it, let's set the root password:
```
passwd
# Type in your root password now
```

And install filesystem tools:
```
emerge sys-fs/dosfstools
emerge sys-fs/e2fsprogs

```

Optional: And install a logging daemon:
```
emerge app-admin/sysklogd
rc-update add sysklogd default`
```

Optional: And a cron daemon, for scheduling commands:
```
emerge sys-process/cronie
rc-update add cronie default
```

Optional: And allow remote access:
```
rc-update add sshd default
```

Optional: And improve file indexing:
```
emerge --ask sys-apps/mlocate
```

## Bootloader

To actually boot the system, we need a bootloader. We'll be using GRUB2:
```
emerge --ask --verbose sys-boot/grub:2
grub-install --target=x86_64-efi --efi-directory=/boot
grub-mkconfig -o /boot/grub/grub.cfg
```

## Configuring Users

Now, we'll install sudo, so that we can run commands as root after installation finishes:
```
emerge app-admin/sudo
```

TODO: Configuration of file *sudoers*

We're now going to add another user, since currently, we are root, which is the only user:
```
useradd -m -G users,wheel,audio -s /bin/bash <username>
passwd <username>
```

Finally, we need to unmount everything and reboot the system. **Replace sdbX with your partitions if needed**:
```
umount /dev/sdb5
umount /dev/sdb4
umount /dev/sdb2
umount -R /mnt/gentoo
```

And now, the big one; rebooting. Make sure you boot into the newly set up hard drive, and not your live image:
```
reboot
```

If you make it past GRUB and to the login prompt, try logging in as the user you added. If you can, try pinging something you consider important. If you can, then **congratulations, everything worked!** The only thing though is that you just have a plain black terminal with white text. The next part of the guide will involve customisation.

## Configuring an X-server

After the reboot, try logging in as a non-root user, to ensure you can do whatever you need. From this point on, the guide will be assuming you are not root.

At this stage, when you log in, you should have a simple black screen, displaying your username and hostname. The next step is to create a graphical environment. This will be achieved by installing what is called *x11* (sometimes just *x*), which is used to draw nice-looking graphics on your screen.

```
sudo emerge --ask --verbose x11-base/xorg-drivers
sudo emerge --ask --verbose x11-base/xorg-server
sudo emerge --ask --verbose x11-apps/xinit
sudo emerge --update --ask --deep --newuse ---with-bdeps=y @world
sudo startx
```


Display manager
```
sudo emerge --ask --verbose x11-misc/sddm
usermod -a -G video sddm
sudo emerge --ask --verbose gui-libs/display-manager-init
sudo emerge --ask --verbose x11-terms/xterm x11-wm/twm x11-apps/xclock
```
FILE /etc/conf.d/display-manager
DISPLAYMANAGER="sddm"
OR IS IT
DISPLAYMANAGER="xdm"


root #rc-update add dbus default
root #rc-update add display-manager default
To start LightDM now:

root #rc-service dbus start
root #rc-service display-manager start

Window manager

```
sudo USE="xinerama" emerge --ask --verbose x11-wm/i3
sudo emerge --ask --verbose x11-misc/dmenu
sudo emerge --ask --verbose sys-apps/accountsservice
sudo USE="pulseaudio" emerge --ask --verbose x11-misc/i3status x11-misc/i3lock
```

Now when you reboot again, you should be brought to a login screen.

## Configuring a Display Manager

TO BE CONTINUED