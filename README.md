# How to install Gentoo Linux (2023 Edition)

![Saint iGNUcius](https://i.imgur.com/V67I2v7.png)

## Table of Contents

- [Preface](#id-preface)
- [Live USB](#id-usb)
- [First Boot](#id-boot1)
- [Disk Preparation](#id-disc)
- [Installing the OS](#id-gentoo)

<div id='id-preface'/>
## Preface

This document is an updated version of a document I wrote in 2021. The aim of this document is to streamline the installation of Gentoo Linux on a new PC. This document is based on information provided by the youtube user [Mental Outlaw](https://www.youtube.com/watch?v=6yxJoMa05ZM&feature=emb_title) and  by the [Gentoo wiki](https://wiki.gentoo.org/wiki/Handbook:AMD64/Full/Installation).

This guide assumes you are installing Gentoo Linux on a new PC or to dual-boot on a non-Linux PC. It also assumes you have a working internet connection.

I will be installing Gentoo Linux on a PC with AMD components. The CPU is an AMD 7950X and the GPU is an AMD 7900XTX. The steps will vary slightly if you are using, for example, an Intel CPU or an NVidia GPU, but the overall process is virtually the same.

```
Any line of code or pseudocode that does not start with a '#' is intended for the reader to execute.
# Any line of code that starts with a '#' is not to be executed by you. This is purely informational.
```

<div id='id-usb'/>
## Live USB

You will need a USB stick with the minimal Gentoo Linux image installed. Find the latest minimal installation CD from [here](https://www.gentoo.org/downloads/) and download it. Then, flash that image to your USB with this command:
```
sudo dd if=<path to ISO file> of=<path to USB device> bs=8192k
sync
# For example, this is what I used:
# sudo dd if=~/Downloads/install-amd64-minimal-20221211T170150Z.iso of=/dev/sdb bs=8192k
# sync
```

On Windows, this can be done using a tool like balenaEtcher.

Once the image is flashed to the USB, plug it into the target PC and boot from the USB.

<div id='id-boot1'/>
## First Boot

When you successfully boot, you will be brought to a black terminal prompt. From here, the first test is to try pinging gentoo.org, to ensure packages can be downloaded as part of the installation. If this fails, please troubleshoot your internet connection.
```
ping -c 10 gentoo.org
```

After confirming your PC has internet access, change the root password to something unique with the following command:
```
passwd
```

<div id='id-disc'/>
## Disk Preparation

Before going any further, think about how you want your files to be stored on your system. For example, my PC has three hard drives: an SSD for system files, an SSD for games and other files, and a HDD for movies, books, and documents. Here is how I will organise my files:

**Primary SSD**
|- /boot partition for booting
|- /grub partition for bootloading
|- /swap partition for hibernation
| - /root partition for system files
**Secondary SSD**
| -/home partition for games and other files
**HDD**
| - /bulk partition for bulk files. Will contain folders such as /books and /movies

All hard drive related commands will use these three hard drives in the examples. To find out the paths to your own hard drives, run the command *lsblk*. For example, here is my output:
```
lsblk
# The next lines are the output. My hard drives are sda, sdb, and nvme0n1 (sdc is the USB we are using). Ignore loop0.
loop0 
sda
sdb
|__sdb1
sdc
|_sdc1
|_sdc2
|_sdc3
|_sdc4
nvme0n1
```

### Wiping the Disks

Run the following command **for each hard drive**:
```
wipefs -a <path to drive>
# Example for my PC
# wipefs -a /dev/sda
# wipefs -a /dev/sdb
# wipefs -a /dev/nvme0n1
```

### Creating Partitions

As mentioned earlier, you must create separate partitions for your data. This is done with the *parted* utility. The following commands are for my PC, and must be adjusted to suit your own file system.
```
# Partition the main drive
parted -a optimal /dev/nvme0n1
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

# Create a 128MiB /boot partition
mkpart primary 3 131
# Name this partition 'boot'
name 2 boot

# Create a 24GB /swap partition
# Substitute (GB of RAM) with the RAM you have
mkpart primary 131 [131 + 1024 * 2 * (GB of RAM) + 1]
# Name this partition 'swap'
name 3 swap

# Fill the rest with the /root partition
mkpart primary [131 + 1024 * 2 * (GB of RAM) + 1] -1
# Name this partition 'root'
name 4 root

# Verify layout with the print command.
# Check everything looks like it is partitioned correctly.
print
quit

# Partition the SSD
parted -a optimal /dev/sda
unit mib
mklabel gpt
mkpart primary 1 -1
name 1 home
print
quit

# Partition the HDD
parted -a optimal /dev/sdb
unit mib
mklabel gpt
mkpart primary 1 -1
name 1 bulk
print
quit
```


You can verify the layout of your drives by running *lsblk* again. If you are not satisfied, you can run the *parted* utility again.

### File System Creation

After partitioning is complete, you must create file systems on your disks to make them useable.

```
# Create a FAT32 file system for the /boot partition
mkfs.fat -F 32 /dev/nvme0n1p2
# Create an ext4 file system for the /root partition
mkfs.ext4 /dev/nvme0n1p4
# Create an ext4 file system for the /home partition
mkfs.ext4 /dev/sda1
# Create an ext4 file system for the /bulk partition
mkfs.ext4 /dev/sdb1
```

The swap partition requires different commands.
```
# Set /swap as a swap partition on the SSD
mkswap /dev/nvme0n1p3
swapon /dev/nvme0n1p3
```

### Mounting the file systems

To use these file systems and install Gentoo, you need to mount the root partition, which means assigning it to a path somewhere on your PC. We will also mount the home partition.
```
mount /dev/nvme0n1p4 /mnt/gentoo
mkdir /mnt/gentoo/home
mount /dev/sda1 /mnt/gentoo/home
mkdir /mnt/gentoo/bulk
```

<div id='id-gentoo'/>
## Installing the OS


Before the next step, make sure your date and time are set correctly. You can check this by running:
```
date
```
If the date and time are not correct, you'll need to set them either:
* Manually, by using the *date* command
* Automatically, by connecting to an ntp server with *ntpd*

Gentoo distributes itself in what it calls a *stage 3 tarball*. This contains the libraries and tools you need to bootstrap the system. It also contains the basis for a particular kind of system, so you have multiple options, such as whether to use openrc or systemd, whether to use a hardened kernel or not, and whether to deny 32-bit libraries.

For this guide, I will be choosing the basic option. Since you don't have a web browser when installing from a live CD, you'll need to go to the [download](https://www.gentoo.org/downloads/#other-arches) page, find the link to the openrc tarball, then paste it into wget. It will probably be the first option listed under *stage archives*. **Replace the link next to wget with the download link for the latest tarball.**
```
cd /mnt/gentoo
wget https://bouncer.gentoo.org/fetch/root/all/releases/amd64/autobuilds/20230101T164658Z/stage3-amd64-openrc-20230101T164658Z.tar.xz
```

The next step is to unpack this tarball.
```
tar xpvf stage3-amd64-20210224T214503Z.tar.xz --xattrs-include='*.*' --numeric-owner
rm stage3-amd64-20210224T214503Z.tar.xz
```

### make.conf

Gentoo uses a package manager and distribution system called *portage*, which allows you to manage packages, install and build packages differently depending on different flags, and change some pre-configured Gentoo settings.

A lot of your customisation will be done from a file called *make.conf*, which is stored in */etc/portage/make.conf* on most Gentoo systems. How you configure this file is entirely system-dependent, depending on any graphics cards you use, what software licenses you want to use, among other things.

For example, there is a variable called *USE* which controls which features packages are built with by default, *VIDEO_CARDS* which controls supported GPUs, and *GRUB_PLATFORMS* to install the correct version of GRUB during this installation.

You can read the man page for make.conf to see every supported flag, which will be useful for tweaking your own installation.

My make.conf file is in this repository. Copy that file to /mnt/gentoo/etc/portage/make.conf and edit it to your liking. You can also manually edit it yourself using my file as a starting point.

### Preparing the Environment

It's almost time to start work inside the new Gentoo environment. Before entering, you'll need to copy some networking settings.
```
cp --dereference /etc/resolv.conf /mnt/gentoo/etc/
```

Next, you need to copy the list of repositories that you'll get everything from.
```
mkdir --parents /mnt/gentoo/etc/portage/repos.conf
cp /usr/share/portage/config/repos.conf /mnt/gentoo/etc/portage/repos.conf/gentoo.conf
```

The next step is mounting all the file systems the kernel will need; /proc/, /sys/, /dev/, and /run.
```
mount --types proc /proc /mnt/gentoo/proc
mount --rbind /sys /mnt/gentoo/sys
mount --make-rslave /mnt/gentoo/sys
mount --rbind /dev /mnt/gentoo/dev
mount --make-rslave /mnt/gentoo/dev
mount --bind /run /mnt/gentoo/run
mount --make-slave /mnt/gentoo/run 
```

Now you need to enter the new environment. After this point, all actions will be performed from inside the new Gentoo environment.
```
chroot /mnt/gentoo /bin/bash
source /etc/profile
export PS1="(chroot) ${PS1}"
```

You'll need to mount the boot directory to perform the next steps:
```
mount /dev/nvme0n1p2 /boot
```

### Packages


**If you significantly mess up any stage beyond this point, you should be able to start again from here.**

The next step is to install a snapshot of the gentoo ebuild repository. This is used by Portage to help build your system. To do this:
```
emerge-webrsync
```

You can now use the *emerge* command-line tool to install packages. The first thing we'll do is install an *ntp* client, so that our system's time is always correct. We'll also make sure this is always done on boot.

Now is also a good time to understand the use flags of packages. When you install a packages with the *--ask* option, it will show you which USE flags are available. Enabled flags are in red, and disabled flags are in blue. It's always good to review the default flags before installing a package, to ensure you're getting the features you want.

```
emerge --ask net-misc/ntp
rc-update add ntp-client default
rc-service ntp-client start
```

Next, we need to choose our *profile*. Think of a profile as a building block for a Gentoo system. It sets how packages are built by default, default compiler options, allowed package versions, etc. It is possible, but non-trivial, to change profiles later, so keep that in mind.

To list all the profiles, do this command. Your output may differ:
```
eselect profile list

Available profile symlink targets:
  [1]  default/linux/amd64/17.1 (stable) *
  [2]  default/linux/amd64/17.1/selinux (stable)
  [3]  default/linux/amd64/17.1/hardened (stable)
...
```

The star means this is your currently selected profile. I will be using the default profile. To select it, I will run:
```
eselect profile set 1
# Or whatever number corresponds to basic profile.
```

After selecting a profile, you must update the system's *@world set*, which is just all the packages a profile uses. You should also run this command if you ever make any changes to the USE flag in your make.conf.

Running the following command will take a long time, so it's the perfect time to have a break. If you want to run it and see what packages would be installed, add the *-p* flag at the end. This way, you can double-check which USE flags these packages will install with. To actually install the packages, remove the *-p*.
```
emerge --ask --verbose --update --deep --newuse [-p] @world
```

Optionally, you can replace the default text editor (Nano) with something else. I prefer using Vim compared to nano, so I will replace Nano with Vim in this step:
```
emerge -q app-editors/vim
```

### Setting your Timezone and Locale

The next step is updating your timezone, to make sure your PC displays the correct time after installation. If you live in a different time zone than I do, please look up what yours is by typing *ls /usr/share/zoneinfo*
```
echo "Pacific/Auckland" > /etc/timezone
emerge --config sys-libs/timezone-data
```

Now we're going to set our locale, which defines how dates are displayed, what alphabet you use, etc. I will be setting a New Zealand locale in the file */etc/locale.gen*. This is the contents of that file:
```
C.UTF8 UTF-8
en_NZ ISO-8859-1
en_NZ.UTF-8 UTF-8
```

Once you've edited your locale, run the following:
```
locale-gen
eselect locale list
# Find your locale in the list. In my case, I want en_NZ.utf8, which has index 6
eselect locale set 6
env-update
source /etc/profile && export PS1="(chroot) ${PS1}"
```

## Kernel configuration

Gentoo encourages users to configure the Linux kernel to their own liking. You could use *genkernel* to automatically configure it, but this guide will opt for manual configuration.

First, download the Linux kernel source code.
```
emerge --ask sys-kernel/gentoo-sources
```

Next, install pciutils, which is required if you are using PCI devices (such as graphics cards).
```
emerge -q sys-apps/pciutils
```

Now, before we go any further, you must know what your PC is like. You need to know what you want, and don't want, to support. Turning off random settings will cause a ton of trouble, so only turn off settings you KNOW you don't want (e.g. nvidia drivers since I'm using an AMD graphics card).

You can find more information about kernel configuration [here](https://wiki.gentoo.org/wiki/Kernel/Gentoo_Kernel_Configuration_Guide). Strictly speaking, kernel configuration is not essential, but it is recommended.

To begin kernel configuration, you need to run this:
```
eselect kernel set 1
cd /usr/src/linux
make menuconfig
```

This will bring you into a large menu, full of options under more options under even more options.

The best thing to do is write down a list of goals you'd like to achieve with your setup, and the check the Gentoo wiki for how you should achieve that. I will do that for my build, so you can follow my example. **[y]** means that the option is built into the kernel during compilation, **[M]** means the option is supported as a module (i.e. the functionality is build into a kernel module that requires explicit loading by the system) and **[n]** means the option is intentionally not built.

**Required Options**

* Device Drivers --->
 * Generic Driver Options --->
  * [y] Maintain a devtmpfs filesystem to mount at /dev
  * [y] Automount devtmpfs at /dev, after the kernel mounted the rootfs
 * SCSI device support  --->
  * [y] SCSI disk support
* File systems --->
 * [y] Second extended fs support
 * [y] The Extended 4 (ext4) filesystem
 * DOS/FAT/NT Filesystems  --->
  * [y] MSDOS fs support
  * [y] VFAT (Windows-95) fs support
 * Pseudo Filesystems --->
  * [y] /proc file system support
  * [y] Tmpfs virtual memory file system support (former shm fs)
 * [y] Network device support --->
  * [y] PPP (point-to-point protocol) support
  * [y] PPP support for async serial ports
  * [y] PPP support for sync tty ports
Don't forget to include support in the kernel for the network (Ethernet or wireless) cards.

**Support for USB input devices**
* Device Drivers --->
 * HID support  --->
  * [y] HID bus support
  * [y] Generic HID driver
  * [y] Battery level reporting for HID devices
  * USB HID support  --->
   * [y] USB HID transport layer
  * [y]USB support  --->
   * [y] xHCI HCD (USB 3.0) support
   * [y] EHCI HCD (USB 2.0) support
   * [y] OHCI HCD (USB 1.1) support

**Support Gentoo-specific features and don't support systemd**
* Gentoo Linux --->
 * [y] Gentoo Linux Support
 * [y] Linux dynamic and persistent device naming (userspace devfs) support
 * [y] Select options required by portage features
 * Support for init systems, system and service managers --->
  * [y] OpenRC, runit and other script based systems and managers
  * [n] systemd 

**Enable multi-core support on CPU**
* Processor type and features  --->
 * [y] Symmetric multi-processing support
  * [y] SMT (Hyperthreading) aware nice priority and policy support
  * [y] Multi-core scheduler support
* Power management and ACPI options  --->
 * [y] ACPI (Advanced Configuration and Power Interface) Support

**Disable wifi support if you decide to not use wifi**
* [y] Networking Support --->
 * Wireless --->
  * [n] Generic IEEE 802.11 Networking Stack (mac80211)
* Device Drivers --->
 * [y] Network Device Support --->
  * [n] Wireless LAN

**Enable kernel virtualisation (KVM) if using virtual machines**
* [y] Virtualization --->
 * [y] Kernel-based Virtual machine (KVM) support
  * [y] KVM for AMD processors support
  
**Options required for VPN usage**
* Device Drivers --->
 * [y] Network device support --->
  * [y] Universal TUN/TAP device driver support

**Options required for AMD 7900 XT graphics card and to disable nvidia/intel features**
* Processor type and features --->
 * [y] MTRR (Memory Type Range Register) support
* Memory management options -->
 * [y] Memory Hotplug -->
  * [y] Allow for memory hot remove
 * [y] Device memory (pmem, HMM, etc...) hotplug support
 * [y] Unaddressable device memory (GPU memory, ...) 
 
* Device Drivers --->
 * Graphics support --->
  * [y] Laptop Hybrid Graphics - GPU switching support
  * [y] Direct Rendering Manager (XFree86 4.1.0 and higher DRI support) --->
  * [y] Enable legacy fbdev support for your modesetting driver
  * [n] ATI Radeon
  * [m] AMD GPU
  *  ACP (Audio CoProcessor) Configuration  ---> 
   * [y] Enable AMD Audio CoProcessor IP support
   * Display Engine Configuration  --->
    * [y] AMD DC - Enable new display engine
    * [y] Enaable HDCP support in DC
    * [y] Enable secure display support
   * [y] HSA kernel driver for AMD GPU devices
  * [n] Intel 8xx/9xx/G3x/G4x/HD Graphics
  * [m] Virtual Box Graphics Card
  * Sound card support  --->
   * Advanced Linux Sound Architecture  --->
     * [y] PCI sound devices --->
     * HD-Audio  --->
     * [y]HD Audio PCI
     * [y] Support initialization patch loading for HD-audio
     * [y] Build HDMI/DisplayPort HD-audio codec support
 * Character devices --->
  * [y] IPMI top-level message handler

**Disable intel microcode because I'm using an AMD CPU**
* Processor types and features --->
 * CPU microcode loading support
  * [n] Intel microcode loading support
  * [y] AMD microcode loading support

**Enable support for PulseAudio**
* Device Drivers --->
 * [y] Sound card support --->
  * [y] Advanced Linux Sound Architecture --->
   * [y] USB Sound Devices --->
    * [M] USB Audio/MIDI driver

**Enable Webcam support**
* Device Drivers  --->
 * [M] Multimedia support  --->
  * Media device types --->
   * [y] Cameras and video grabbers
  * Media drivers --->
   * [y] Media USB Adapters  --->
    * [M] USB Video Class (UVC)
    * [y] UVC input events device support

**Enable support for Wii-U Gamecube Controller Adapter**
* Device Drivers --->
 * HID support --->
  * Special HID drivers --->
   * [M] Nintendo Wii / Wii U peripherals

**Enable support for Xbox 360 wireless receiver**
* Device Drivers
 * Input device support
  * [y] Generic input layer (needed for keyboard, mouse, ...)
   * [y] Joysticks/Gamepads
    * [y] X-Box gamepad support
     * [y] X-Box gamepad rumble supportThursday, 05. January 2023 12:38AM 


**Allow execution of 32-bit binaries**
* Processor types and features --->
 * [n] Machine Check / overheating reporting
* Binary Emulations --->
 * [y] IA32 Emulation

**Enable support for GPT labels**
* Enable the block layer --->
  * Partition Types --->
   * [y] Advanced partition selection
    * [y] EFI GUID Partition support

**Preparing for the  AMD drivers**

* Device Drivers --->
 * Firmware Drivers -->
  * [y] Mark VGA/VBE/EFI FB as generic system framebuffer
 * Graphics support --->
 * Frame buffer Devices -->
  * Support for frame buffer devices -->
   * [y] EFI-based Framebuffer Support
  * [y] VGA Arbitration

This next step is only necessary for AMD. The steps are different for nvidia.
```
emerge -av linux-firmware
emerge -av media-libs/mesa
emerge -av dev-libs/rocm-opencl-runtime
```

After configuration, we build the kernel.
```
make -j$(nproc)
make modules_install
make install
```

### Creating /etc/fstab

All partitions of your system must be listed inside the file */etc/fstab*. To configure this file, you will need to find the *UUID* of each partition, which is a unique identifier for that partition. To find the UUIDs, run this command:
```
blkid
```
This repository contains my /etc/fstab file. You can use it as an example to learn how yours should be set up.

### Networking Configuration

Optional: Change your PC's name by editing */etc/conf.d/hostname*. This isn't an IP address; it's a nice name like "GentooBox" or "MyPC".

Now we're going to configure a DHCP client. Please note, this guide will NOT cover configuring wi-fi, since I won't be using it on my PC.

First, download this:
```
emerge --ask --noreplace net-misc/netifrc
```
Next, we're going to specify which interface we're using. In my case, I'm using a wired connection over the interface *enp3s0*. 
To find which interface you should use, run *ifconfig* or *ip addr* and find the interface with your PC's IP address on it. 
Open the file */etc/conf.d/net*, and write the following, replacing <interface> with your interface:
```
config_<interface>="dhcp"
```

Next we will make sure the system runs DHCP at boot:
```
emerge net-misc/dhcpcd
cd /etc/init.d
ln -s net.lo net.<interface>
rc-update add net.<interface> default
```

### Other Steps

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

Optional: Install a logging daemon:
```
emerge app-admin/sysklogd
rc-update add sysklogd default`
```

Optional: Install a cron daemon, for scheduling commands:
```
emerge sys-process/cronie
rc-update add cronie default
```

Optional: Allow remote access over SSH:
```
rc-update add sshd default
```

Optional: Improve file indexing:
```
emerge --ask sys-apps/mlocate
```

### Bootloader

To actually boot the system, we need a bootloader. We'll be using GRUB2:
```
emerge --ask --verbose sys-boot/grub:2
grub-install --target=x86_64-efi --efi-directory=/boot --removable
grub-mkconfig -o /boot/grub/grub.cfg
```

### Configuring Users

We need to install *sudo*, so that we can use root commands as a user.
```
emerge --ask --verbose sudo
```

Next, we're now going to add another user, since currently, we are root, which is the only user:
```
useradd -m -G users,wheel,audio -s /bin/bash <username>
passwd <username>
```

You can do this for as many users as you feel like. Each one will create a directory in the */home* path called */home/<username>*, which is used to store that user's files.

Then, we need to edit the file */etc/sudoers*. Look in that file and follow the instructions of this line:
```
## Uncomment to allow any user to run sudo if they know the password
## of the user they are running the command as (root by default)
# Defaults targetpw   #Ask for the password of the target user
#  ALL ALL=(ALL) ALL  # WARNING: only use this together with 'Defaults targetpw'
```

## Rebooting at the end

Finally, we need to unmount everything and reboot the system:
```
exit
cd /
umount -l /mnt/gentoo/dev{/shm,/pts,}
umount -R /mnt/gentoo
```

And now, the big one; rebooting. Make sure you boot into the newly set up hard drive, and not your live image:
```
reboot
```

If you make it past GRUB and to the login prompt, try logging in as the user you added. If you can, try pinging something you consider important. If you can, then **congratulations, everything worked!** However, you only have a plain black terminal with white text. The next part of the guide will involve beautifying your system.

## Configuring an X-server

After the reboot, try logging in as a non-root user, to ensure you can do whatever you need. From this point on, the guide will be assuming you are not root.

At this stage, when you log in, you should have a simple black screen, displaying your username and hostname. The next step is to create a graphical environment. This will be achieved by installing what is called *x11* (or just *x*), which is used to draw nice-looking graphics on your screen. There is also a modern alternative called Wayland, but this guide doesn't cover that.

```
# Install all the x11 stuff
sudo emerge --ask --verbose x11-base/xorg-drivers x11-base/xorg-server x11-apps/xinit x11-apps/xrandr

# You also need a terminal emulator, which will display your command lines. I willl install Kitty.
sudo emerge --ask dev-vcs/git x11-terms/kitty
```

## Configuring a Display Manager

A display manager is what it sounds like. It controls what's displayed on screen. It also provides a graphical login prompt when you start your system, hosts your wallpapers, etc. Basically, it make you switch from being a command line user to a desktop user.

The display manager I'm installing is **SDDM**. Here's how it's installed:
```
sudo emerge --ask --verbose gui-libs/display-manager-init x11-misc/sddm
sudo usermod -a -G video sddm
```

You then need to configure OpenRC, telling it to run your display manager. Part of this is editing the file */etc/conf.d/display-manager*, adding the following lines:
```
CHECKVT=7
DISPLAYMANAGER="sddm"
```

Afterwards, run the following commands:
```
sudo rc-update add dbus default
sudo rc-update add display-manager default
```

## Configuring a Window manager

A window manager controls how your screen is split up. I will be using dwm, which is a tiling window manager. if you're doing the same, replace '6.0' with whatever file is generated by the package.

```
sudo emerge --ask --verbose x11-wm/dwm
sudo ln -s /etc/portage/savedconfig/x11-wm/dwm-6.0 /etc/portage/savedconfig/x11-wm/dwm-6.0.h
sudo emerge --ask --verbose x11-misc/dmenu 
```

We also need to do this:
```
sudo gpasswd -a <user> video
```


## The End

And just to be safe, one final update:
```
sudo emerge --update --ask --deep --newuse ---with-bdeps=y @world
```

Now, when you restart, everything should be ready to go.

Welcome to Linux! [This is your life now.](https://www.youtube.com/watch?v=ezUoiaoQCTs)


