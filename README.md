# So you want to install Gentoo?

![Saint iGNUcius](https://i.imgur.com/V67I2v7.png)

My aim is to make the installation of a (spefically my) Gentoo system relatively painless. This will start out as a series of instructions, but maybe some day I'll create some scripts to help you better. Probably not though.

This information will be mostly taken from the excellent youtube video by [Mental Outlaw](https://www.youtube.com/watch?v=6yxJoMa05ZM&feature=emb_title), but more generalised than the video he made for his own PC. When I say generalised, I mean that I'll do this for my own PC,  but I'll explain each step along the way.

This guide will be for x86_64 PCs (which Gentoo calls amd64, and is almost certainly what your computer's architecture is). More details can be found in the [Gentoo AMD64 Handbook](https://wiki.gentoo.org/wiki/Handbook:AMD64) if you desire. 

***Disclaimer:*** I am not responsible for bricking your system. That's your fault for following the advice of some stranger on the internet.

## Getting Started

If you're not already running Linux, find a live image, which is where you'll be performing the installation from. This image needs burned onto a USB flash drive or a CD (I recommend the flash drive). I recommend doing this with the Balena Etcher tool, which provides a simple and clean interface for flashing images.

If you are installing Gentoo on a system that's already running Linux, and it's being installed on a new hard drive, you can perform the same installation process from your host machine. Just know that you'll have to run the commands as root by using **sudo**, instead of just running them from the live image where you are root by default.

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

This will show you all the disks on your system, as well as how it partitioned. Find the disks that correspond to your hard drives. **Make sure you don't mix these up, or you'll format the wrong hard drive**, which I have done once. It's a painful experience.

## Preparing the Media

Hard drives are split into *partitions*, which are logical divisions between areas of the drive. These partitions are used by the system for different purposes. To properly perform this step, you will need to know how each hard drive will be used (i.e. are you going to use it for mass storage or to install the O.S.).

I will be using one SSD for the OS, and two HDDs for storage. Here is how I plan to organise my system. This guide will be setting exactly this up, but your steps can be modified depending on your hardware and use case.

You must also check if your motherboard uses BIOS or UEFI, because this affects how you create these partitions and what file systems you'll be using. My PC (as well basically all modern PCs) use UEFI, so please check the Gentoo wiki for what to do if you're using BIOS.

- 128GiB SSD
|- 2MiB /grub partition, used by the *GRUB* program to bootload the system
|- 128MiB /boot partition, used by Linux during booting.
|- 24GiB /swap partition, used to hibernate the system, and is used when the system runs out of RAM.
|- rest is the /root partition, which is the core of the system.
- 2TiB HDD consisting of one large /home partiton, which is occupied by user files.
- 3TiB HDD consisting of one large partition called /bulk, which I will use for storing large files like games and movies.


To make sure the hard drive is completely unpartitioned before installation, run this command, **replacing /dev/sdX with whatever your hard drives are.**  You can find these paths with the *lsblk* command. Once you know, run these commands to purge the drives:

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

# Create a 128MiB /boot partition
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
name 1 home
print
quit

# Partition the 3TiB HDD
parted -a optimal /dev/sdc
unit mib
mklabel gpt
mkpart primary 1 -1
name 1 bulk
print
quit
```

## Creating the File Systems

All your partitions will need the correct file systems on them. 
Basically you want to use FAT32 for boot, and ext4 for everything except 
the swap partition.
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
wget https://bouncer.gentoo.org/fetch/root/all/releases/amd64/autobuilds/20211128T170532Z/stage3-amd64-openrc-20211128T170532Z.tar.xz
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

My make.conf file is in this repository. Copy that file to /mnt/gentoo/etc/portage/make.conf and edit it to your liking. You can also manually edit it yourself using my file as a starting point.

## Preparing the Environment

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

Anyway, you'll need to mount the boot directory to perform the next steps:
```
mount /dev/sda2 /boot
```

**If you significantly mess up any stage beyond this point, you should be able to start again from here.**


## Emerge

The next step is to install a snapshot of the gentoo ebuild repository. This is used by Portage to help build your system. To do this:
```
emerge-webrsync
```

You can now use the *emerge* command-line tool to install packages. The first thing we'll do is install an *ntp* client, so that our system's time is always correct. We'll also make sure this is always done on boot.

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

## Setting your Timezone and Locale

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

Gentoo encourages users to configure the Linux kernel to their own liking. Don't worry, it sounds harder than it is. You could use *genkernel* to automatically configure it, but this guide will opt for manual configuration.

First, download the Linux kernel source code.
```
emerge --ask sys-kernel/gentoo-sources
```

Next, install pciutils, which is required if you are using PCI devices (such as graphics cards).
```
emerge -q sys-apps/pciutils
```

Now, before we go any further, you must know what your PC is like. You need to know what you want, and don't want, to support. Turning off random settings will cause a ton of trouble, so only turn off settings you KNOW you don't want (e.g. AMD drivers since I'm using an Nvidia graphics card).

You can find more information about kernel configuration [here](https://wiki.gentoo.org/wiki/Kernel/Gentoo_Kernel_Configuration_Guide). Strictly speaking, kernel configuration is not essential, but it is recommended.

For this guide, I will show what I configured, which you can use as a basis. To begin kernel configuration, you need to run this:
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
  * [y] KVM for Intel (and compatible) processors support

**Options required for VPN usage**
* Device Drivers --->
 * [y] Network device support --->
  * [y] Universal TUN/TAP device driver support

**Options required for GTX970 graphics card and to disable AMD features**
* Processor type and features --->
 * [y] MTRR (Memory Type Range Register) support
* Device Drivers --->
 * Graphics support --->
  * [n] Nouveau (NVIDIA) cards
  * Frame buffer devices --->
   * [y] Support for frame-buffer devices --->
    * [n] nVidia Framebuffer Support
    * [n] nVidia Riva support
 * Character devices --->
  * [y] IPMI top-level message handler

**Disable AMD microcode because I'm using an Intel CPU**
* Processor types and features --->
 * CPU microcode loading support
  * [n] AMD microcode loading support

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
     * [y] X-Box gamepad rumble support
     * [y] LED Support for Xbox 360 controller 'BigX' LED

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

**Preparing for the  nVidia drivers**

* Device Drivers --->
 * Graphics support --->
  * [y] VGA Arbitration
  * Frame buffer devices --->
   * [y] Support for frame-buffer devices --->
    * [n] Simple framebuffer support
 * Generic Driver Options --->
  * Firmware loader --->
   * (nvidia/gm204/acr/bl.bin nvidia/gm204/acr/ucode_load.bin nvidia/gm204/acr/ucode_unload.bin nvidia/gm204/gr/fecs_bl.bin nvidia/gm204/gr/fecs_data.bin nvidia/gm204/gr/fecs_inst.bin nvidia/gm204/gr/fecs_sig.bin nvidia/gm204/gr/gpccs_bl.bin nvidia/gm204/gr/gpccs_data.bin nvidia/gm204/gr/gpccs_inst.bin nvidia/gm204/gr/gpccs_sig.bin nvidia/gm204/gr/sw_bundle_init.bin nvidia/gm204/gr/sw_ctx.bin nvidia/gm204/gr/sw_method_init.bin nvidia/gm204/gr/sw_nonctx.bin) Build named firmware blobs into the kernel binary

After configuration, we build the kernel.
```
make -j4
make modules_install
make install
```

## Creating /etc/fstab

All partitions of your system must be listed inside the file */etc/fstab*. To configure this file, you will need to find the *UUID* of each partition, which is a unique identifier for that partition. To find the UUIDs, run this command:
```
blkid
```
This repository contains my /etc/fstab file. You can use it as an example to learn how yours should be set up.

## Networking Configuration

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

## Bootloader

To actually boot the system, we need a bootloader. We'll be using GRUB2:
```
emerge --ask --verbose sys-boot/grub:2
grub-install --target=x86_64-efi --efi-directory=/boot --removable
grub-mkconfig -o /boot/grub/grub.cfg
```

## Configuring Users

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

# Rebooting at the end

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
sudo emerge --ask --verbose x11-base/xorg-drivers x11-base/xorg-server x11-apps/xinit

# You also need a terminal emulator, which will display your command lines. I willl install XTerm.
sudo emerge --ask --verbose x11-terms/xterm
echo 'XTerm*Background: Grey7' >>~/.Xresources
echo 'XTerm*Foreground: DeepSkyBlue3' >>~/.Xresources
sudo xrdb ~/.Xresources
```

## Configuring a Display Manager

A display manager is what it sounds like. It controls what's displayed on screen. It also provides a graphical login prompt when you start your system, hosts your wallpapers, etc. Basically, it make you switch from being a command line user to a desktop user.

The display manager I'm installing is **SDDM**. Here's how it's installed:
```
sudo emerge --ask --verbose gui-libs/display-manager-init x11-misc/sddm
usermod -a -G video sddm
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

We also need to ins
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
