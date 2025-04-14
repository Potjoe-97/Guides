
## Disclaimer

I'm not an engineer, I do not take responsibility for bricked devices, I barely understand what I'm doing. Take this tutorial as a testimony of my own (successful) experience. I do not guarantee I'll be able to answer your questions either. Use at your own risks. 

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

*   Extract the `super` partition & get information about its structure with `imjtool`. In my case, `imjtool` is located in the mtkclient folder:

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


We can see here that the `super` image:

*   Has 3 slots: `default`, `main_a`, and `main_b`
*   `main_a` and `main_b` each contain 3 partitions: `product`, `system`, and `vendor`
*   Some partitions in `main_b` are empty: `product_b` and `vendor_b`

We'll work on the `main_a` slot, since it's the only complete group containing phone data.

### Replace system_a with Lineage OS

A new folder has appeared: `extracted`. It contains the extracted images from `super.bin`. There are 6 files, corresponding to the 6 partitions:

*   product_a.img
*   product_b.img
*   system_a.img
*   system_b.img
*   vendor_a.img
*   vendor_b.img

We want to replace the system partition with the Lineage OS ROM, remember? In the `extracted` folder, delete `system_a.img` (since we determined that slot _a_ is the complete one), and move the Lineage OS 20 ROM into the folder.

> [!WARNING]
> Make sure to extract the downloaded Lineage OS archive: the file extension must be .img, not .img.gz

Then rename the lineage OS ROM to “system_a.img”. At this point, we've replaced the system partition with Lineage OS, and we just need to rebuild the flashable partition `super_new.img`.

### Build the new super_new.img

To do this, we need to rebuild it identically to the original image, as its size must match the phone’s partition.

*   In the Debian terminal, navigate to the backup folder.
*   Note the image size with the command:

```
stat -c '%n %s' super.bin
```

Result:

```
super.bin 10468982784
```

The size of my original image is `10468982784` (yours may differ!).

Do the same for all new images:

*   In the Debian terminal, go to the “extracted” folder
*   Check the size of the different images with:

```
stat -c '%n %s' *.img
```

Result:

```
product_a.img 1101447168
product_b.img 0
system_a.img 2806325248
system_b.img 147349504
vendor_a.img 404115456
vendor_b.img 0
```

Now rebuild the image using lpmake. Before launching the command, make sure to adjust each device/group/partition’s size as shown below:

```
lpmake --metadata-size 65536\
 --metadata-slots=3\
 --device super:10468982784\
 --group=main_a:4311887872\
 --group=main_b:147349504\
 --partition=product_a:readonly:1101447168:main_a\
 --partition=product_b:readonly:0:main_b\
 --partition=system_a:readonly:2806325248:main_a\
 --partition=system_b:readonly:147349504:main_b\
 --partition=vendor_a:readonly:404115456:main_a\
 --partition=vendor_b:readonly:0:main_b\
 --image=product_a=product_a.img\
 --image=system_a=system_a.img\
 --image=system_b=system_b.img\
 --image=vendor_a=vendor_a.img\
 --sparse --output ./super_new.img
```

⚠️ The numbers in this example are **not universally valid**. I recommend preparing the command in Wordpad, then running it. Adjust all numbers to your image sizes.

*   `--metadata-slots`: Must match the number of slots on the device. We had this number thanks to imjtool 
*   `--device super`: The size of the `super` partition on the device. It must match exactly: in our case, '10468982784'.
*   `--group main_a`: Sum of all partition file sizes in group main_a. Eg: product_a + system_a + vendor_a = 1101447168 + 2806325248 + 404115456 = 4311887872
*   `--group main_b`: Sum of all partition file sizes in group main_b.
*   `--partition`: File sizes with permission (`readonly`).  Adjust file sizes with results you got from last step.
*   `--image`: Path to each partition image, except empty ones. In our case, no vendor_b.img or product_b, we know they are empty. 

Run the command and be patient. If you get a header magic error, grab a coffee and wait.

You should now have a new `super_new.img` file in the 'extracted' folder.

## Flash the phone

### Unlock bootloader & flash

First, unlock the phone’s bootloader.

*   On the phone, go to Settings, enable Developer Options. Then enable USB Debugging.
*   Connect the phone via USB. In a Windows terminal, run:

```
adb reboot fastboot
```

This will boot the phone into fastboot mode.

*   On the phone, select “Reboot to bootloader” and click the endcall key to validate. You should see “=> FASTBOOT mode…”
*   Then run:

```
fastboot flashing unlock
```

*   On the phone, press “Volume Up” to confirm. Your bootloader should now be unlocked.
*   Reboot to fastboot again, running :

```
fastboot reboot fastboot
```
*   Select “Reboot to bootloader” and click the endcall key to validate. You should see “=> FASTBOOT mode…”

You can now flash the image we built earlier:

```
fastboot flash super super_new.img
```

Then:

```
fastboot reboot
```

ET VOILA! You flashed a Custom GSI ROM to your Doov R17 Pro!

> [!WARNING]
> **DO NOT RELOCK THE BOOTLOADER** after flashing or you will get a dm-verity corruption warning.

## Root with Magisk (optional)

Re-enable Developer Options & USB Debugging in Lineage OS.

In the Windows terminal, in the folder where you downloaded the Magisk apk, run:

```
adb install Magisk-v28.1.apk
```

Then follow the official Magisk install instructions: [Installation | Magisk](https://topjohnwu.github.io/Magisk/install.html). Grab your boot.img from the backup folder and upload it to the phone, then patch it with Magisk.

If you have boot_a.bin and boot_b.bin, upload `boot_a.bin`.

Since the phone has no recovery partition, and patching init_boot didn't work, I only patched and flashed the boot partition. Make sure to flash the patched image to `boot_a`, like this:

```
fastboot flash boot_a /path/to/magisk_patched_[random_strings].img
```

I didn’t flash `vbmeta`; maybe you should? I don’t know! It worked for me by only flashing `boot_a`.

Don’t forget to:

```
fastboot reboot
```

after flashing, then get back to magisk instructions.

## Lineage OS settings

*   Go to Settings > Phh Treble Settings > Misc features
    *   Enable "Rotation perf hint instead of touch"
    *   Enable “Mediatek GED Kpi support”
    *   To disable navbar & gestures (for keypad + touch navigation only), enable “Force navigation bar disabled”
*   Go to Settings > Phh Treble Settings > IMS features. Emergency calls didn't work without these options for me.
    *   Click “Create IMS APN”
    *   Click “Install IMS APK for Mediatek S vendor”
    *   Enable Request IMS network
*   For t9 typing, I recommend installing TT9 from the playstore.
*   I recommend Button mapper with following parameters :
    *   To enable the D-pad center button, assign "One Click" to "D-Pad Center"
    *   To enable Call button, assign "One Click" to "Personalized keycode 5"
    *   To enable Star button (useful for tt9)  assign "One Click" to "Personalized keycode 17"

## Credits & sources

[🤔 binboupan's blog | Taking Control of the Xiaomi Qin F22 Pro](https://binboupan.github.io/2023/08/qin-f22-pro/)

[Patching Dynamic Partitions in Android Super Image · senyuuri's blog](https://blog.senyuuri.info/posts/2022-04-27-patching-android-super-images/)

[android_device_Unihertz_Atom_LXL/docs/HOW-TO-FLASH-SUPER.md at master · ADeadTrousers/android_device_Unihertz_Atom_LXL](https://github.com/ADeadTrousers/android_device_Unihertz_Atom_LXL/blob/master/docs/HOW-TO-FLASH-SUPER.md)
