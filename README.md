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
# ping -c 10 www.gentoo.org
```

If the pings do not work, then you have a problem. Good luck fixing that, because I'm not going to. If the pings works, you can continue reading.

The next step is finding the target medium for installation, such as an internal hard drive (which this guide will assume). To find your hard drive, run:
```
# fdisk -l

Disk /dev/sdb: 1.84 TiB ...
(You'll find lines like this in the output)
```

This will show you all the disks on your system, as well as how it partitioned. Find the disk that corresponds to your hard drive (e.g. my hard drive is */dev/sdb*). **Make sure you don't mix these up, or you'll format the wrong hard drive** (which I'm ashamed to admit I've done in the past).

## Preparing the Hard Drive

If your hard drive is not already partitioned, this step will show you how. To make sure the hard drive is completely unpartitioned before installation, run this command, **replacing /dev/sdb with whatever your hard drive is**:
```
# wipefs -a /dev/sdb
```

Once that's done, we'll need to use the *parted* program to format the hard drive, which just means we're going to split it into logically-divided parts. To begin, run this, **replacing /dev/sdb with whatever your hard drive is**:
```
# parted -a optimal /dev/sdb
```

TO BE CONTINUED

## Portage

## Make.conf

## Users

## Display