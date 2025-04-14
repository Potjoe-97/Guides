
## Disclaimer

I'm not an engineer, I do not take responsibility for bricked devices, I barely understand what I'm doing. Take this tutorial as a testimony of my own (successful) experience. I do not guarantee I'll be able to answer your questions either.

## Prerequisites

I'm writing this tutorial for a Windows machine, but most tools are Linux-based, so it shouldn't be too hard to adapt. Follow the installation instructions for each program (Read the Readme!)

_To flash Lineage OS 20 (Android 13)_

*   adb  
*   [MTKClient](https://github.com/bkerler/mtkclient) (don't forget to install drivers)  
*   [Python 3](https://www.python.org/downloads/)  
*   Linux machine – or [Windows Subsystem for Linux](https://apps.microsoft.com/detail/9msvkqc78pk6?hl=fr-FR&gl=FR) (I'm using Debian)  
*   [imjtool - A tool for unpacking Android boot and system images](https://newandroidbook.com/tools/imjtool.html)  
*   [LineageOS 20](https://sourceforge.net/projects/andyyan-gsi/files/lineage-20-td/) rom. **ALWAYS** download the arm64 build, not the a64 one! I picked the bgN variant: it comes with Google services and does not have root by default.  
*   [lpunpack & lpmake](https://github.com/Exynos-nibba/lpunpack-lpmake-mirror)

_To gain root access_

*   Latest [Magisk](https://github.com/topjohnwu/Magisk/releases/) apk

## Backup the phone

We want a full backup of the phone before doing anything, since only a self-made backup can ensure the phone can be restored with IMEI information. Do not rely on ROMs found online, make your own backup!

We'll use mtkclient for that.

*   In a Windows terminal, in the mtk folder, run the following command to start the backup:

```
.\mtk.bat rl --skip userdata backup
.\mtk.bat r preloader preloader.bin --parttype=boot1
```

*   With the phone turned off, plug it in: when the battery logo appears, unplug it, then plug it in again immediately.  
*   Let the backup finish: it's very long, and that's normal.

The backup files (= firmware) are now in a folder called `backup`.

## Build a new flashable image

To flash a phone, you need to replace the original `system` partition with the Lineage OS ROM. However, there are two issues:

*   The Doov R17 Pro uses A/B partitioning (so there are two system partitions, `system_a` and `system_b`)  
*   Both system partitions are contained within a single `super` partition.

You cannot flash Lineage OS directly over a system partition that's embedded inside a `super` partition. So we must rebuild the `super` partition to include the Lineage OS ROM, then flash the entire `super` partition. The tools we’ll use are `imjtool`, `lpunpack` (optional), and `lpmake`.

### Extract the super partition

*   Open a Debian terminal and navigate to the mtkclient folder with the `cd` command. In my case:

```
cd "/mnt/c/Users/potjoe/git/mtkclient"
```

*   Extract the `super` partition & get information about its structure with imjtool. In my case, imjtool is located in the mtkclient folder:

```
./imjtool.ELF64 backup/super.bin extract
```

Here's the result:

```
MMapped: 0x7f63a4913000, imgMeta 0x7f63a4914000
liblp dynamic partition (super.img) - Blocksize 0x1000, 3 slots
LP MD Header @0x3000, version 10.2, with 6 logical partitions @0x0 on block device of 9984 GB, at partition super, first sector: 0x800
Partitions @0x3100 in 3 groups:
        Group 0: default
        Group 1: main_a
                Name: product_a (read-only, Linux Ext2/3/4/? Filesystem Image, @0x100000 spanning 1 extents of 1 GB) - extracted
                Name: system_a (read-only, Linux Ext2/3/4/? Filesystem Image, @0x41c00000 spanning 1 extents of 1 GB) - extracted
                Name: vendor_a (read-only, Linux Ext2/3/4/? Filesystem Image, @0xc1a00000 spanning 1 extents of 385 MB) - extracted
        Group 2: main_b
                Name: product_b (read-only,  empty) - extracted
                Name: system_b (read-only, Linux Ext2/3/4/? Filesystem Image, @0xb8d00000 spanning 1 extents of 140 MB) - extracted
                Name: vendor_b (read-only,  empty) - extracted
```

...

(Due to length limits, the full content will be written to the file.)
