## rpi-clone
Version 2 is a complete rewrite with improved capability over
the original.  See the examples below.

rpi-clone is a shell script that is for cloning a running
Raspberry Pi booted source disk (SD card or USB disk) to a destination
disk which will be bootable. Destination disks are SD cards in the SD
card slot or a USB card reader, USB flash disks, or USB hard drives.

rpi-clone may work in SD card booted devices other than a
Raspberry Pi because when initializing a disk, rpi-clone images a
first /boot partition and boot loader setup can be captured.
But this will depend on how the boot loading is handled on each device.

I also am now using rpi-clone on my Debian desktop, but there are too many
variables in how a /etc/fstab can be set up and a desktop bootloader like
grub can be configured for this to be an officially  supported way of
using rpi-clone.

#### Clone by initialization
Source disk mounted partition types are compared to
corresponding destination disk partitions.
If the types are not compatible, then the clone is an
initialization.  First, the destination partition structure is
initialized to match the source disk.  This is is a convenience that gets
the destination disk partitioned so you can avoid manual partitioning.
All partitions are then cloned either
by imaging source unmounted partitions to corresponding destination
partitions or by doing a destination mkfs followed by a file system
sync of source mounted partitions to the destination partitions.
So to avoid file system inconsistencies, live partitions are synced
and not imaged with one exception.  If the first partition
is the /boot partition, it is imaged so that bootloader install state
can be preserved.  This is not an issue on a Pi where the GPU knows how
to boot, but could be on other systems that have a bootloader install.
A mounted /boot is rarely active so its file system
state should be consistent, just don't be doing anything to modify your
boot configuration when running rpi-clone.

#### Clone by syncing
If the file system types
are compatible, the destination partitions will be mounted and the clone
is a sync of modified files from source to destination.  After an
initialize clone, subsequent clones will be syncs.  You can skip
the initialize clone and go straight to a sync clone
if a destination disk is manually partitioned and file
systems created (mkfs) that match the mounted source partitions.  In
this case a destination disk does not need all partitions to match, only
the mounted ones.  Doing this you can have special case use of partitions on
different systems.  See my Pi3 example below.


## Install
rpi-clone is on github and is downloaded by cloning the repository.
It is a standalone script and the install is a simple copy to a
bin directory.  When run it checks its program dependencies and offers to
install needed packages.  But currently rpi-clone knows how to install
only Debian packages with apt-get.

#### On a Raspberry Pi:
```
	$ git clone https://github.com/billw2/rpi-clone.git 
	$ cd rpi-clone
	$ sudo cp rpi-clone rpi-clone-setup /usr/local/sbin
```
Run rpi-clone or rpi-clone-setup with no args to print usage.

rpi-clone-setup is for setting the hostname in /etc/hostname and /etc/hosts
files.  It is run automatically by rpi-clone if -s args are given,
but before your first clone using a -s option, test run rpi-clone-setup with:
```
      $ sudo rpi-clone-setup -t testhostname
```
And check the files under /tmp/clone-test to be sure the files have been
edited correctly.  If you need additional customizations to a clone,
add them to the rpi-clone-setup script.

#### On other OS:
  To install on another OS, rpi-clone may be renamed to suit.  For example,
  on my Debian desktop I rename:
```
	$ git clone https://github.com/billw2/rpi-clone.git 
	$ cd rpi-clone
	$ sudo cp rpi-clone /usr/local/sbin/sys-clone
	$ sudo cp rpi-clone-setup /usr/local/sbin/sys-clone-setup
```
  
If your other OS is a SD card booted system, it will possibly work.
However it currently does not work for emmc booted devices.

rpi-clone does not directly support usage on a desktop OS.
However, I do use it with my Debian desktop because my setup script
handles my /etc/grub.d/ custom menus and fstab, and the script runs
grub_install.  rpi-clone does handle editing of
PARTUUID values in /etc/fstab, but a customized setup script for
a desktop might need to handle file system UUID values or device
name editing in /etc/fstab and the bootloader config.  If these possible
issues are handled in a setup script, then rpi-clone should work fine
creating clone backup disks for a desktop.

## Usage
To get a usage screen showing available options,
run rpi-clone without any arguments:
```
pi@rpi0: $ sudo rpi-clone
No destination disk given.

usage: rpi-clone sdN {-v|--verbose} {-f|--force-initialize} {-f2}
         {-u|--unattended} {-U|--Unattended} {-q|--quiet}
         {-s|--setup} {-e|--edit-fstab sdX }
         {-m|--mountdir dir } {-l|--leave-sd-usb-boot}
         {-a|--all-sync} {-F|--Force-sync} {-x} {-V|--version}
         {--convert-fstab-to-partuuid}

    -v      - verbose rsync, list all files as they are copied.
    -f      - force initialize the destination disk by imaging the booted disk
                partition structure.  File systems are then synced or imaged.
    -f2     - force initialize only the first 2 partitions to the destination.
                So a multi partition USB boot can initialize clone back to
                a 2 partition SD card.
    -u      - unattended clone if not initializing.  No confirmations asked,
                but abort if disk needs initializing or on error.
    -U      - unattended even if initializing. No confirmations asked,
                but abort only on errors.
    -q      - quiet mode, no output unless errors or initializing. Implies -u.
    -s host - add 'host' to args passed to script rpi-clone-setup and run it
                after cloning but before unmounting partitions. For setting
                clone disk hostname, but args can be what the script expects.
                You can give multiple "-s arg" options.
    -e sdX  - edit destination fstab to change booted device names to new
                device 'sdX'.  This is Only for fstabs that use device names.
                Used for setting up a USB bootable disk.
    -m dir  - Add dir to a custom list of mounted directories to sync.  Then
                the custom list will be synced instead of the default of all
                mounted directories.  The root directory is always synced.
                Not for when initializing.
    -l      - leave SD card to USB boot alone when cloning to SD card mmcblk0
                from a USB boot.  This preserves a SD card to USB boot setup
                by leaving the SD card cmdline.txt using the USB root.  When
                cloning to USB from SD card this option sets up the SD card
                cmdline.txt to boot to the USB disk.
    -a      - Sync all partitions if types compatible, not just mounted ones.
    -F      - force file system sync even if errors.
                If source used > destination space error, do the sync anyway.
                If a source partition mount error, skip it and do other syncs.
    -x      - use set -x for very verbose bash shell script debugging
    -V      - print rpi-clone version.
```
+ See examples below for usage of these command line options.
+ rpi-clone version 1 briefly had a -s option that is replaced with a
  -s option that has different meaning.
+ **--convert-fstab-to-partuuid** converts the booted fstab from using device
names to PARTUUID.  This is a helper if you wish to convert to PARTUUID as
is standard in recent Raspbian distributions.  After running, PARTUUID
usage will propagate to subsequent clones.  This changes the booted fstab
and cmdline.txt, so have a backup first.
+ FUSE mounts (ssh mounts) should be unmounted before cloning or else the
directory mounted on will not stat and the directory will not be made on the
clone.  You will get a readlink stat error from rsync because root can't access
a users FUSE mount - only the user can.

## rpi-clone Example Runs
#### 0) Examples review - a quick guide to what the examples cover in detail.
1. Typical two partition clones - SD card or USB disk to USB disk:
```
	$ rpi-clone sda
```
2. USB boot - clone back to SD card slot:
```
	$ rpi-clone mmcblk0
```
3. Clone to USB disk intended for use as a standalone Pi3 bootable
disk.  No special rpi-clone args are required.
4. SD card to USB disk clone to create a SD card to USB boot setup:
```
If fstab uses PARTUUID:
	$ rpi-clone -l sda
If fstab uses device names:
	$ rpi-clone -l sda -e sda
```
5. USB boot clone back to SD card slot that preserves SD card to USB boot setup:
```
	$ rpi-clone -l mmcblk0
```
6. Attempted clone to a disk that is too small.
7. Manually partition a disk with three partitions so it can
be cloned to from a two partition boot.
8. Clone from three partition disk to smaller disk large enough
to hold the source three partitions.
```
	$ rpi-clone sdb
```
9. Clone from three partition disk to smaller disk (sdN or mmcblk0)
not large enough to hold the source three partitions.
A first initialize clone forces a clone of only the first two partitions:
```
	$ rpi-clone sdb -f2
```
10. Subsequent sync clones from three partition disk to two partition
disk (sdN or mmcblk0):
```
If the source third partition is not mounted:
	$ rpi-clone sdb
If the source third partition is mounted, select partitions to clone:
	$ rpi-clone sdb -m /boot
```
**Note** - if a larger USB disk is manually partitioned to create more than
three partitions as in example 7, a smaller disk can be initialize
cloned to only if it is large enough to hold at least part of the last
source partition.
If it is not, then the clone will have to be to a two partition -f2
clone or a clone to a manually partitioned destination.  So, for a
multi partition disk, select partition number and sizes with a eye
towards how you will be cloning back to smaller disks.

#### 1) First clone to a new SD card in USB card reader
In this example a new SD card in a USB card reader has been plugged in
that I want to clone to, but it could also be to a USB disk or from a
USB disk back to the SD card slot.
In this case, the disk showed up as sdb because I have another USB
disk sda plugged in. Look in /proc/partitions to see where yours is.
The destination disk does not have partition types matching the booted disk.
+ The clone will be an initialize because of partition types mismatch.
+ The destination last partition will be resized down in this case because
the destination disk is smaller than the booted disk.
+ rpi-clone will ask for a destination root label which I will give
so I can keep track of my clones.
+ If PARTUUID is used in fstab and cmdline.txt, those files will be edited
to use the PARTUUID of the destination SD card.  The SD card will
bootable when plugged in to the SD card slot.
+ If fstab and cmdline.txt use device names (mmcblk0), then rpi-clone
does need to edit and the card will be bootable when plugged
into a SD card slot.
```
pi@rpi0: $ sudo rpi-clone sdb

Booted disk: mmcblk0 16.0GB                Destination disk: sdb 8.0GB
---------------------------------------------------------------------------
Part      Size    FS     Label           Part   Size    FS     Label
1 /boot   58.4MB  fat16  --              1       8.0GB  fat32  --
2 root    16.0GB  ext4   SD-RPI-s1                               
---------------------------------------------------------------------------
== Initialize: IMAGE mmcblk0 partition table to sdb - FS types mismatch ==
1 /boot     (22.5MB used)    : IMAGE     to sdb1  FSCK
2 root      (6.0GB used)     : RESIZE(8.0GB) MKFS SYNC to sdb2
---------------------------------------------------------------------------
Run setup script       : no
Verbose mode           : no
-----------------------:
** WARNING **          : All destination disk sdb data will be overwritten!
                       : The partition structure will be imaged from mmcblk0.
-----------------------:

Initialize and clone to the destination disk sdb?  (yes/no): yes
Optional destination rootfs /dev/sdb2 label (16 chars max): SD-RPI-8a
... 
```

#### 2) Subsequent clone to the same SD card in USB card reader as example 1
This time the destination partition type will match the source booted
types, and I'll add a rpi-clone-setup script -s arg to set a
different destination disk hostname.

+ The clone will be a pure sync where only modified files will be copied.
+ The setup script will set the hostnames in the destination disk files
/etc/hostname and /etc/hosts to what I give with -s, in this case rpi2.
 ```
pi@rpi0: $ sudo rpi-clone sdb -s rpi2

Booted disk: mmcblk0 16.0GB                Destination disk: sdb 8.0GB
---------------------------------------------------------------------------
Part      Size    FS     Label           Part   Size    FS     Label
1 /boot   58.4MB  fat16  --              1      58.4MB  fat16  --
2 root    16.0GB  ext4   SD-RPI-s1       2       8.0GB  ext4   SD-RPI-8a
---------------------------------------------------------------------------
== SYNC mmcblk0 file systems to sdb ==
/boot       (22.5MB used)    : SYNC to sdb1 (58.4MB size)
/           (6.0GB used)     : SYNC to sdb2 (8.0GB size)
---------------------------------------------------------------------------
Run setup script       : rpi-clone-setup  rpi2
Verbose mode           : no
-----------------------:

Ok to proceed with the clone?  (yes/no): 
```

#### 3) Cloning a Pi3 when fstab uses PARTUUID
If fstab and cmdline.txt use PARTUUID as is the case in recent
Raspbian distributions, rpi-clone always edits**
the destination fstab and cmdline.txt to use the PARTUUID of the
destination disk.  So the destination is always bootable.  If it
is a USB flash or hard drive it is automatically bootable on a Pi3
as a USB disk so long as the Pi3 has been USB boot enabled with
a program_usb_boot_mode=1 line in /boot/config.txt.

** There is one exception.  When using the -l option, which is used for
creating or preserving a special SD card to USB boot, the cmdline.txt
on the SD card is not edited after a clone to the SD card, see
examples 4 and 5.

#### 4) Creating a USB bootable disk for other than a USB enabled Pi3
rpi-clone can be used to create a SD card to USB boot setup and preserve
that setup when cloning from a USB boot back to the SD card slot.
With the SD card booted and a target USB disk plugged in and assuming
the USB disk shows up as sda, the initial clone command depends on
fstab usage of device names or PARTUUID.

=> Before you do this, have a backup of your booted SD card made
as in example 2 without the -l option because these steps will
change the booted SD card cmdline.txt to a USB boot.

If fstab is using PARTUUID, run:
```
$ rpi-clone -l sda
```
Or if fstab is using device names, run:
```
$ rpi-clone -l -e sda sda
```
+ Destination disk "sda" will be synced or initialized if required (or add
the -f option to force initialize).
+ After files are synced the destination sda fstab and cmdline.txt will
be edited to reference either device names or PARTUUID for the USB disk.
For the fstab uses device names case, the "-e sda" means to edit the
destination /etc/fstab to use "sda" for the root (will be sda1) and
/boot (will be sda2) lines. Also, the destination disk /boot/cmdline.txt
will be edited to use root=/dev/sda2.  It is expected that when the USB disk
is plugged in for booting to, it will be sda and this will be a cause
of boot failure if it is not.  So using PARTUUID is better because that
will reliably boot.
+ The -l option causes the SD card cmdline.txt to be backed up to
cmdline.boot and the destination USB disk cmdline.txt to be copied
to the SD card.  Since the USB cmdline.txt was edited to reference
the USB disk, the next Pi boot will start with the SD card
/boot partition, but will redirect to using the USB root partition.
Since the USB fstab was edited to reference the USB disk, the Pi will boot
with the USB partition 1 mounted on /boot.
The SD card /boot partition that initiated the boot process
is no longer in use but can remain in place for subsequent
SD card to USB boots.  To make the SD card standalone bootable
again, its cmdline.boot can be moved back to cmdline.txt.

+ If -l is not used, rpi-clone will not replace the currently booted SD card
cmdline.txt and it will need to be edited by hand for the USB boot to work.

+ Also a caution note if fstab uses device names: check your
/boot to be sure it is mounted with /dev/sda1 after booting to USB.
I have a Pi where this fails even though syslog says it mounted.
Just be sure to check when you first do this and before you try example 5.

Now when the Pi is booted from SD card to USB and the SD card is no longer
in use, the SD card slot is available for cloning to.

#### 5) Cloning back to SD cards in the SD card slot from USB boots
Whether the boot was a Pi3 straight to USB or a SD card to USB,
the SD card is not in use so it is free to clone back to.  This
creates a standalone bootable SD card:
```
$ rpi-clone mmcblk0
```
However, for the case where the boot was SD card to USB,
this destroys the ability of the SD card to boot to USB.
To preserve that SD to USB boot setup, run:
```
$ rpi-clone -l mmcblk0
```
+ The SD card is cloned to as before.  It now has the USB /boot/cmdline.txt.
+ But the -l option prevents editing that cmdline.txt to reference the SD card.
It is left alone so that it still references the USB root partition.
So the clone has created USB disk to SD card backup  while preserving
the SD card to USB boot setup.  On the SD card a backup cmdline.boot
is created and edited to reference the SD card.  That backup can be moved
to be cmdline.txt to make the SD card standalone bootable should
you ever want to do that.
Or you could just clone to the SD card without using -l.
+ Both above mmcblk0 clone commands apply whether using PARTUUID or
device names.  When using device names and cloning to SD cards,
rpi-clone knows fstab device names need editing so "-e mmcblk0p" is assumed.
Now the SD card can be left in permanently and periodically cloned to for
backups and reboots to USB will work as you want.  Or other SD
cards can be inserted to create a set of backups.
If making a clone for another Pi that will be SD card bootable, don't use -l.
+ Warning: this works if the original SD card to USB boot setup has edited
the USB /etc/fstab to reference USB partitions as is done by rpi-clone
when creating a USB bootable disk with -l.  If you have an existing
SD card to USB boot setup where this was not done, then your USB boot
likely has the SD card /boot partition mounted, the SD card is in use
and using rpi-clone for a clone back to the SD card slot will not work.


#### 6) Clone to smaller 4GB SD card
I happen to have an old 4GB SD card and here's a try to clone to it: 
```
root@rpi2: ~$ rpi-clone sda

Booted disk: mmcblk0 15.8GB                Destination disk: sda 4.0GB
---------------------------------------------------------------------------
Part      Size    FS     Label           Part   Size    FS     Label
1 /boot   58.4MB  fat16  --              1      58.4MB  fat16  --
2 root    15.8GB  ext4   SD-RPI-16N      2       3.9GB  ext4   --
---------------------------------------------------------------------------
== SYNC mmcblk0 file systems to sda ==
/boot       (22.5MB used)    : SYNC to sda1 (58.4MB size)
/           (5.9GB used)     : SYNC to sda2 (3.9GB size)
---------------------------------------------------------------------------
Run setup script       : no
Verbose mode           : no
-----------------------:
** FATAL **            : Partition 2: source used > destination space.
-----------------------:

Aborting!
  Use -F to override used > space fail.
```
So even if rpi-clone thinks that the sync won't work because of lack of
space, there is a -F option which will allow the clone to proceed
anyway.  The interesting thing about this case is that while this might
seem a bad idea, the sync will actually come close to succeeding.  That's
because the root used space includes a 1.8GB file system based
swap file (/var/swap) that will be excluded from the sync.  If this
clone is forced with -F, the card may boot, but there could be some missing
files if the rsync runs out of space and fails to complete and some things
would not work.
This is just a FYI.


#### 7) Clone SD card to USB disk with extra partitions
If you have space with a larger USB disk, you can manually partition it
with extra partitions and clone to it.  If partition types and file systems
are made to match the booted SD card, then rpi-clone will sync files and
not try to initialize so the extra destination partitions will not be
touched.

The requirement to make this work is getting the first two partition types
and file systems right, but the sizes may be different. gparted will make
filesystems automatically but cfdisk or fdisk will not and if
file systems aren't made, rpi-clone will fail to mount the partitions.

For this example I wanted to partition a 64GB flash disk into 3 partitions
so I could have a large third partition for data.
I made the second root partition 16GB so I could clone this to a
32GB disk and still have a data partition.

If using cfdisk or fdisk to make the partitions and then making the file
systems, the work would be:
```
  Partition Type               Size          Make File System
  1: type c  W95 FAT32 (LBA)   100MiB        mkfs -t vfat -F 32 /dev/sda1
  2: type 83 Linux              16GiB        mkfs.ext4 /dev/sda2
  3: type 83 Linux             rest of disk  mkfs.ext4 /dev/sda3
```
But what I did was use gparted so the file systems were made for me.
Also, in anticipation of initialize cloning back to SD cards,
I set the first partition start to be 8192 (by setting "Free space preceding"
to 4MiB) to match Raspbian distribution SD card images.
Also I made partition sizes in multiples of 4MiB for SD card compatibility.

Now I cloned to the 64GB disk.  It synced only my two booted partitions
instead of initializing, and it left the third partition alone:
```
pi@rpi2: ~$ sudo rpi-clone sda

Booted disk: mmcblk0 15.8GB                Destination disk: sda 64.2GB
---------------------------------------------------------------------------
Part      Size    FS     Label           Part   Size    FS     Label
1 /boot   58.4MB  fat16  --              1     104.4MB  fat32  --
2 root    15.8GB  ext4   SD-RPI-16N      2      16.8GB  ext4   Samsung 64GB A
                                         3      47.3GB  ext4   --
---------------------------------------------------------------------------
== SYNC mmcblk0 file systems to sda ==
/boot       (22.5MB used)    : SYNC to sda1 (104.4MB size)
/           (5.9GB used)     : SYNC to sda2 (16.8GB size)
---------------------------------------------------------------------------
Run setup script       : no
Verbose mode           : no
-----------------------:

Ok to proceed with the clone?  (yes/no):
```
This was a boot enabled Pi3, so I simply powered down, pulled the SD
card, and rebooted into the three partition USB disk.


#### 8) Clone 64GB USB disk with extra partitions to smaller 32GB USB disk
With the USB disk made in example 7 booted and the third partition mounted,
this is a clone to a smaller 32GB USB disk that is still large enough
to hold three partitions.  The disk is not manually formatted so it will
be an initialize clone.  The space used in the 64GB source third partition
fits into the size of the destination 32GB disk third partition,
so there is no problem:
```
pi@rpi2: ~$ sudo rpi-clone sdb

Booted disk: sda 64.2GB                    Destination disk: sdb 31.5GB
---------------------------------------------------------------------------
Part               Size    FS     Label           Part   Size    FS     Label
1 /boot           104.4MB  fat32  --              1      31.5GB  fat32  --
2 root             16.8GB  ext4   Samsung 64GB A                          
3 /home/pi/media   47.3GB  ext4   --                                      
---------------------------------------------------------------------------
== Initialize: IMAGE sda partition table to sdb - FS types mismatch ==
1 /boot               (21.5MB used)  : IMAGE     to sdb1  FSCK
2 root                (5.9GB used)   : MKFS SYNC to sdb2
3 /home/pi/media      (54.3MB used)  : RESIZE(14.6GB) MKFS SYNC to sdb3
---------------------------------------------------------------------------
Run setup script       : no
Verbose mode           : no
-----------------------:
** WARNING **          : All destination disk sdb data will be overwritten!
                       :   The partition structure will be imaged from sda.
-----------------------:

Initialize and clone to the destination disk sdb?  (yes/no): 
```
Note that if I had partitioned the 64GB disk with more than three
partitions it would have been more difficult to clone down to the
32GB card.  If there had been 4 partitions, then a smaller disk has
to be large enough to image the sizes of the first three source partitions.
If the disk is too small for that, then an initialize clone would be
limited to a two partition clone as the next example shows.  The other
alternative would be a manual partition.  The take away is that you need
to consider how you would be cloning to smaller disks when you partition
a larger disk for a Pi.

#### 9) Clone 64GB USB disk with extra partitions to new 16GB SD card
With a USB boot, the SD card slot is available for use, so I plugged in
a 16GB SD card to clone to:
```
pi@rpi2: ~$ sudo rpi-clone mmcblk0

Booted disk: sda 64.2GB                    Destination disk: mmcblk0 15.8GB
---------------------------------------------------------------------------
Part               Size    FS     Label           Part   Size    FS     Label
1 /boot           104.4MB  fat32  --              1      15.8GB  fat32  --
2 root             16.8GB  ext4   Samsung 64GB A                       
3 /home/pi/media   47.3GB  ext4   --                                   
---------------------------------------------------------------------------
Initialize required    : partition - types mismatch.
                       :   The minimum destination disk size is 16.9GB
                       :   The destination disk is too small.
                       :   You could try a two partition -f2 clone.
-----------------------:
```
This failed because there is a type mismatch that requires an initialize
and additionally the SD card does not have the space for three partitions
given the size of the source disk second partition.  The solution is to
tell rpi-clone to clone only the first two partitions and accept that
this backup cannot back up the data partition.  The -f2 option is just
for going back to a two partition disk from a multi partitioned disk:
```
pi@rpi2: ~$ sudo rpi-clone mmcblk0 -f2

Booted disk: sda 64.2GB                    Destination disk: mmcblk0 15.8GB
---------------------------------------------------------------------------
Part               Size    FS     Label           Part   Size    FS     Label
1 /boot           104.4MB  fat32  --              1      15.8GB  fat32  --
2 root             16.8GB  ext4   Samsung 64GB A                          
3 /home/pi/media   47.3GB  ext4   --                                      
---------------------------------------------------------------------------
== Initialize: IMAGE sda partition table to mmcblk0 - forced by option ==
1 /boot               (21.5MB used)  : IMAGE     to mmcblk0p1  FSCK
2 root                (5.9GB used)   : RESIZE(15.7GB) MKFS SYNC to mmcblk0p2
---------------------------------------------------------------------------
-f2                    : force initialize to first two partitions only
Run setup script       : no
Verbose mode           : no
-----------------------:
** WARNING **          : All destination disk mmcblk0 data will be overwritten!
                       :   The partition structure will be imaged from sda.
-----------------------:

Initialize and clone to the destination disk mmcblk0?  (yes/no): 
```
I'm using PARTUUID in /etc/fstab, but if I weren't, this clone would also
automatically edit mmcblk0p names into the destination disk fstab.


#### 10) Sync Clone 64GB USB disk with extra partitions to 16GB SD card
With an initialize clone to the SD card done in example 8, I expect
subsequent clones to be sync clones.  But I run the rpi-clone command and
I get an error requiring another initialize.  This time the
error is because rpi-clone wants to clone the mounted third partition and
there is no destination third partition:
```
pi@rpi2: ~$ sudo rpi-clone mmcblk0 

Booted disk: sda 64.2GB                    Destination disk: mmcblk0 15.8GB
---------------------------------------------------------------------------
Part               Size    FS     Label           Part   Size    FS     Label
1 /boot           104.4MB  fat32  --              1     104.4MB  fat32  --
2 root             16.8GB  ext4   Samsung 64GB A  2      15.7GB  ext4   SD-16c
3 /home/pi/media   47.3GB  ext4   --                                      
---------------------------------------------------------------------------
Initialize required    : partition 3 /home/pi/media - destination missing.
                       :   Unmount source partitions or use -m
-----------------------:
```
But I want to sync and not do another long -f2 initialize,
so there are two choices.
If I unmount the third partition, the clone will sync.  If I don't want
to do that, I can tell rpi-clone to sync clone only the /boot partition
(the root partition is included by default):
```
pi@rpi2: ~$ sudo rpi-clone mmcblk0 -m /boot

Booted disk: sda 64.2GB                    Destination disk: mmcblk0 15.8GB
---------------------------------------------------------------------------
Part               Size    FS     Label           Part   Size    FS     Label
1 /boot           104.4MB  fat32  --              1     104.4MB  fat32  --
2 root             16.8GB  ext4   Samsung 64GB A  2      15.7GB  ext4   SD-16c
3 /home/pi/media   47.3GB  ext4   --                                      
---------------------------------------------------------------------------
== SYNC sda file systems to mmcblk0 ==
/boot                 (21.5MB used)  : SYNC to mmcblk0p1 (104.4MB size)
/                     (5.9GB used)   : SYNC to mmcblk0p2 (15.7GB size)
---------------------------------------------------------------------------
Run setup script       : no
Verbose mode           : no
-----------------------:

Ok to proceed with the clone?  (yes/no): 
```

## Author
Bill Wilson
billw--at--gkrellm.net
