# Linux storage notes

1. [Disks and partitions](#1-disks-and-partitions)
2. [Filesystems](#2-filesystems)

## 1 Disks and partitions

### 1.1 Intro to local storage

Local storage means any storage devices plugged directly into the server (called internal drive)
- typically disk drives, SSD flash drives, dvd drives, blu-ray drives etc
- drives are normally plugged using SATA, SAS or SCSI adapters to the back of the server
- opposite of local storage _networked storage_, NAS and SAN are two major forms of networked storage

### 1.2 Storage capacity units

VMware and Linux count storage capacity in slightly different ways
- VMware uses the standard _gigabyte_
- Linux uses the more correct _gibibyte_

Decimal / Base10
- 1 kilobyte (KB) = 1000 bytes
- 1 Megabyte (MB) = 1000 kilobytes
- 1 Gigabyte (GB) = 1000 megabytes
- 1 Terabyte (TB) = 1000 gigabytes

Binary
- 1 kibibyte (KiB) = 1024 bytes
- 1 Mebibyte (MiB) = 1024 kibibytes
- 1 Gibibyte (GiB) = 1024 mebibytes
- 1 Tebibyte (TiB) = 1024 gibibytes

Hard disk capacities are described and marketed by drive manufacturers using the standard metric definition of the gigabyte, but when a 400 GB drive's capacity is displayed by Linux, it is reported as 372 GiB using a binary interpretation.

With small numbers, the differences are not huge until you reach Terabyte or Petabyte territory.
- 1 Petabyte (PB) = 0.88 Pebibytes (PiB)
- 1.1 Petabyte (PB) = 1 Pebibyte (PiB)
  - a difference of more than 129 TB!

### 1.3 Devices and device files

Almost everything in linux is represented as a file. Memory, terminal, disk drives are all represented as files.

Example - the terminal is represented as a file
- ``tty`` tells us that the terminal is represented by the file ``/dev/pts/1``
- ``ls -l /dev/pts`` to see ``crw--w----. 1 root tty 136, 1 Mar 14 21:16 1``
  - first letter is ``c`` means the file is a character device file
- ``echo "See I'm not a file. I'm really your terminal" > /dev/pts/1`` will print the text to the screen
  - this is because the file just represents the terminal we are running now
- compare with ``echo "See I'm not a file. I'm really your terminal" > /temp/tempfile`` which will write the text to the regular file called ``/temp/tempfile``!
- ``rm /tmp/tempfile`` to clean up

Local storage devices are represented as files in the ``/dev`` directory - short for devices.
- ``ls -l /dev | grep sd`` to see all that start with ``sd``
  - ``sd`` stands for scsi drive, but covers sas and sata drives too
  - can sometimes see devices start with ``hd`` if they are old drives connected using old adapters
- we see 3 devices: ``sda``, ``sda1``, ``sda2``. 
  - The device is named ``a`` and the number at the end is the partition number on that device

Use ``fdisk`` to see partitions on the disk
- ``fdisk -cu /dev/sda``
  - ``c`` option switches off DOS compatibility mode
  - ``u`` option gives us sectors rather than cylinders when talking about size.
    - sector size normally 512 bytes or half a kb. Better report as Mb, Gb etc.
- type ``p`` to print the partition table to screen
  - we see 2 partitions, ``sda1`` and ``sda2``, with the first as a boot partition
  - hard to understand the numbers
- type ``q`` to quit out the fdisk program

Use ``parted`` to see partitions
- ``parted``
- type ``p`` to print partition table
  - easier to understand print out then fdisk :)
  - can see one partition (/dev/sda1) is used as the boot partition, at 500mb
  - the other partition (/dev/sda2 )is used for LVM, taking up the rest of the capacity

Running ``ls -l /dev | grep sd``, you get 
- ``brw-rw----. 1 root disk 8, 0 Mar 13 21:40 sda``
  - the entire device
- ``brw-rw----. 1 root disk 8, 1 Mar 13 21:40 sda1``
  - partition one on the device
- ``brw-rw----. 1 root disk 8, 2 Mar 13 21:40 sda2``
  - partition two on the device

The ``b`` means _block device file_ used for drives, as opposed to a character device file.
- character device files read and write one character at a time and does not suport random access
- block device files supports random access.
  - never write directly into a block device file which can cause data corruption
  - when the kernel gets read or write request, it will pass it over to the driver for that device
  - each device file has a major device number and a minor device number
    - in the ``ls`` output above, ``8,`` is the major number and ``0`` is the minor number
    - major tells the kernel which driver to use
    - minor tells the kernel which partition of a device to use

### 1.4 Adding block storage devices

Add 2 new hard discs to either the physical machine or the VM you are running
- ``ls -l /dev | grep sd`` will immediately pick up the new devices
  - ``brw-rw----. 1 root disk 8, 16 Mar 13 21:40 sdb``
  - ``brw-rw----. 1 root disk 8, 32 Mar 13 21:40 sdc``
  - Linux auto-recognises it as a scsi device ``sd``
    - the devices gets labeled alphabetically, ``b`` and ``c`` based on the order the device is discovered when scanning the scsi controllers
  - the major versions are the same - so same driver is used as before
  - the minor numbers are 16 and 32 because historically in linux, each device can have 16 partitions.
    - first device has minor numbers 0 to 15
    - second device has 16 - 31, etc etc.

``udev`` is a device manager that runs as a daemon on our system listening for device state changes issued by the kernel
- when devices are added or removed, ``udev`` updates block device files in the the device directory ``/dev``
- ``tail /var/log/messages`` - can see logs of devices being added and removed
- supposing ``sdb`` was removed and we reboot the computer, the previous ``sdc`` would be relabelled as ``sdb``.
  - no guarantee that the block device file name refers to the same device
  - highly recommended to use the device's UUID, which doesn't change, instead of the filename

### 1.5 Partitions

Disks can be carved into partitions of different sizes and the partitions do not have to use up all the available space.

There are two types of partitions
- primary
- extended
  - modified primary partition that acts as a container that can hold many partitions

In the beginning, all partitions were primary partitions.
- _master boot record_ (MBR), which holds the partition table, only allows 4 primary partitions
- now, we have the extended partitions so we are not limited to only 4.

MBR partition scheme: 
- only 4 primary partitions ``/dev/sda1``, ``/dev/sda2``, ``/dev/sda3``, ``/dev/sda4``
- at the beginning of the disk is a record called the label, or master boot record (MBR), containing the partition table
  - the partition table is only big enough to store 4 partitions

With extended partitions, one of the 4 partitions in the MBR is used as the extended partition
- the extended part can have many more numbers of _logical partitions_
  - so if we wanted 6 partitions on a drive, we'd create 3 primary partitions, 1 extended partition with 3 logical partitions
- best practice is to put the boot partition in a primary partition

A file system doesn't span multiple partitions - file systems are written to partitions in a one-to-one manner, like discs

A further partition system - _GUID partition table_ (GPT)
- a huge shortcoming of the MBR partition system is that it can't handle drives larger than 2TB
- GPT can support drives up to Zettabytes in size, and supports up to 128 partitions
- to boot from a GPT drive, the system must support Unified Extensible Firmware Interface (UEFI) boot
- use ``parted`` command to work with GPT partitions, NOT ``fdisk``
  - warning, ``parted`` makes changes as you type unlike ``fdisk`` which only makes changes when you explicitly commit

GPT partition scheme
- _Legacy MBR header_ occupies the first sectors of the device, allows some backward compatibility
- _Primary GPT_, where definitions of up to 128 partitions are held
- the partitions make up the main bulk
- at the end of the drive is a backup copy of the _Primary GPT_ incase the first copy gets corrupted

### 1.6 Managing partitions

``ls -l /dev | grep sd`` returns all our drives and partitions
- ``brw-rw----. 1 root disk 8, 0 Mar 13 21:40 sda``
- ``brw-rw----. 1 root disk 8, 1 Mar 13 21:40 sda1``
- ``brw-rw----. 1 root disk 8, 2 Mar 13 21:40 sda2``
- ``brw-rw----. 1 root disk 8, 16 Mar 13 21:40 sdb``
- ``brw-rw----. 1 root disk 8, 32 Mar 13 21:40 sdc``

Warning, getting it wrong has serious consequences of corrupting or losing data.

``fdisk`` is an common, interactive partition management tool
- no changes are actually made until you issue the ``w`` command to write changes to disk

#### 1.6.1 Creating a partition using fdisk

``fdisk -cu /dev/sdb``
- type ``p`` to print partitions, there are none
- type ``n`` to add a new partition
  - prompted primary or extended - ``p`` for primary
  - prompted for partition number (1-4) - ``1``
  - prompted for first sector - input the first that fdisk offers us - ``2048`` here
  - prompted for last sector - lets make it 50Mb - ``+50M`` 
    - ``G`` for gigabytes
    - if you dont specify ``M`` or ``G``, ``fdisk`` will assume you mean sectors
- type ``p`` to print new partition (not actually written to disk yet)
- type ``w`` to commit changes
  - writes and exits the ``fdisk`` interactive terminal
- ``partprobe`` - some folk run this to force the kernel to re-read the partition table
- ``parted /dev/sdb``
  - type ``p`` to show partition table and our newly created partition
  - we see that there is still free space in that device, allowing us to create more, extend the partition or leave it unused.
  - type ``q`` to exit ``parted`` interactive terminal

#### 1.6.2 Changing partition type using fdisk

The partition was created with the default partition type is ``83 Linux``

``fdisk /dev/sdb``
- type ``l`` to see a long list of partition types with their codes expressed in hexadecimal notation
  - notable partition types are
    - ``7 HPFS/NTFS``
    - ``82 Linux swap / So``
    - ``83 Linux``
    - ``8e Linux LVM``
    - ``fd Linux raid auto``
- type ``p`` to print existing partition
- type ``t`` to change partition type
  - it'll output the partition number we've selected - important if we have many
  - prompted to exter partition type hex code - ``8e`` for Linux LVM
- type ``w`` to commit or type ``q`` to exit without making changes

#### 1.6.3 Deleting a partition using fdisk

Be very careful you don't delete the wrong partition. 
- deleting a partition takes any filesystem and data with it
- wise to take backups and double check before proceeding

``fdisk /dev/sdb``
- type ``p`` to print partitions
- type ``d`` to delete
  - it'll show the selected partition
- type ``p`` to show the updated partition table
- type ``w`` to commit only when you are sure

``fdisk -lcu /dev/sdb`` 
- ``l`` tells fdisk to list it without going into non-interactive mode

#### 1.6.4 Creating a GPT partition using parted

``parted /dev/sdb``
- ``mklabel gpt`` to make a GTP partition table
  - it'll warn that data will be lost - say ``yes``
- ``p`` to see that the partition table is not ``gpt``
- ``unit MB`` specify unit is MB
- ``mkpart primary 0MB 50MB`` make a primary partition from 0MB to 50MB
  - it'll warn of improper partition alignment - type ``ignore`` for now
- ``p`` to see the created partition

#### 1.6.5 Deleting a GPT partition using parted

``parted /dev/sdb``
- type ``rm 1`` to remove partition 1
- type ``p`` to see that its gone

``parted`` has no safety net like ``fdisk``. It will go ahead and remove it the moment you press ``return` 

### 1.7 Final notes on partitions

If you have a single disk on your computer, it is common to put ``boot`` and ``swap`` in separate partitions.
- Other than that, people often put ``temp`` and ``var`` on separate partitions. 
- This is to stop these areas filling up and using up all the space on your disk. They will only fill up to the allocated partition space

Other notes
- Put ``boot`` into a primary partition
- Use ``GPT`` partitions instead of MBR, esp for larger disks
- Dont write directly into block device files, esp the case for the file for the entire drive, as it doesnt - understand partitions - so can write across all partitions

## 2 Filesystems

### 2.1 Intro to filesystems

Database and swap can write to devices and partitions without filesystems - a raw partition
- most other applications require a filesystem

Each partition gets its own filesystem in a one-to-one manner, and they can be different

E.g.
- ``ext4`` to partition 1
- ``XFS`` to partition 2
- ``reiserfs`` to partition 3

Different file system work better than others for different requirements.

The process of writing a filesystem to a partition is sometimes called "formatting", but in reality theres no low-level formatting happening.
- instead, the partition gets marked with filesystem structures and metadata 
- e.g. start sector, end sector, and number of inodes
- this differs from a low-level factory format of a drive

Some databasees write to raw disk because
- performance - not a huge diff, but removing filesystem layer could make milliseconds
- caching - system memory used first in a filesystem before lazily writing to disk. Databases may have their own caching logic
- simplied support - from the POV of the database so they dont have to support all the filesystems out there.
- more control over I/O

For the most part, applications sit on top of file systems.

### 2.2 Creating filesystems

To see list of kernel supported fs, ``cat /proc/filesystems``
- the ones with ``nodev`` in first column is not currently in use.

``fdisk -l /dev/dsc`` - check if it has partition (nope!)
- ``fdisk /dev/sdc`` - to create one
- ``n`` for new partition
- ``p`` for primary partition
- ``1`` for partition number
- default
- ``p`` to print and double check
- ``w`` to commit

``mke2fs -t ext4 -n /dev/sdc1`` write filesystem ext in partition
- ``t`` option means type
- ``n`` option means do a dry run without actually going ahead
- default no label, 
- block size 4096 bytes, which system decides automatically. can specify with ``b`` option
- 196608 inodes - files can be written into filesystem, can specify with ``I`` option
- superblock locations

``mke2fs -t ext4 -I 123456789 /dev/sdc/1``
- fails, invalid inode size, can only be between 128 and 4096 bytes
- mke2fs has validation to stop us doign silly stuff, but it doesnt stop everything

``mke2fs -t ext4 /dev/sdc1``
- the defaults are defined in ``etc/mke2fs.conf``
- journaled filesystem is is one with a log to assist in recovery

### 2.3 Manual mounting

Mounting is like assigning the filesystem an address so it can be accessed. Can be done manually or automatically via configuration.

``mkdir /testmount`` create a _mountpoint_ to mount it to.

``mount -t ext4 /dev/sdc/1 /testmount`` command is used for manual mounting a drive to a mountpoint
- ``t`` option is for filesystem type
- ``mount -t xfs /dev/sdc1 /testmount`` will get a filesystem error as device is not xfs

```mount -l -t ext4`` list all mounts of type ext4 to check it has worked
- ``l`` option is for listing

``cp -r /var/log /testmount`` copy all log files over to mountpoint
- ``ls /testmount`` to see all the log files

``umount /testmount`` to unmount 
- we lose access to all the files we just copied
- can specify mount point or device
- cant unmount if it is in use, e.g. if our shell is in that location
  - ``lsof /testmount`` to see processes using device
  - ``fuser -cuv /testmount`` see users using /testmount
    - ``c`` lists current dir
- lazy umount using ``-l`` option which wil unmount when its no longer in use
- ``ls /testmount`` to see there are no files there

``mkdir -p /1/2/3/4/5/6`` create a new directory to use as mountpoint that already has a file in it!
- ``touch ./tremendousfile``
- ``echo 'yeeha' > ./tremendousfile``
- ``cat ./tremendousfile``

``mount -t ext4 /1/2/3/4/5/6``
- ``ls /1/2/3/4/5/6`` - should see all the log files, but no sign of ``tremendousfile`` we just created
- ``ls /1/2/3/4/5/6 | grep trem`` - see nothing

``umount /1/2/3/4/5/6``
- ``ls /1/2/3/4/5/6`` - and we can see our ``tremendousfile`` again!

### 2.4 Mount options and automatic mounting

Options to specify when mounting
- discard option ``mount -t ext4 -o discard /dev/sdc1 /testmount``
  - discard option makes filesystem thin provisioning aware. It adds some intell to the mounted filesystem so when we delete data, the storage subsystem (normally a shared storage array on the back-end) is told about the delete, so it can reclaim the deleted space and use it for the volumes. On backend storage array this keeps thin volume thin, allowing other volumes to use the space
- ``mount -lt ext4`` to see the discard option

Automatic mounting
- ``umount /dev/sdc1`` - unmount
- ``mount`` to see we have a clear system

``vim etc/fstab``
- add new line for mount entry ``/dev/sdc1   /testmount    ext4    ro,discard     0   0``
  - device, mountpoint, filesystem, options, dumping, fscking
  
instead of ``/dev/sdc1``, better to specify with UUID found using below commands
- ``bldid`` use block id command to get uuid
- ``lsblk --fs`` to print devices and fs with uuid
- replace ``/dev/sdc/1`` with ``UUID=<uuid>``

We can now reboot to test automatic mounting.
- instead of rebooting, we can just run ``mount -a``, which is just what rebooting does anyway
- ``mount -lt ext4`` to see it mounted

Now that it is in ``etc/fstab`` we can now just mount by specifying mountpoint or device 
- ``mount /testmount`` or ``mount /dev/sdc1`` without specifying options. 
- mount will check fstab for its options

### 2.5 Superblocks and inodes

Superblock contains metadata about the filesystem, structural information
- Each unix system has at least one superblock. Most keep several copies in case of lose or corruption.
- mounting or accessing any files requires the superblock
- superblock kept in memory for fast access to mounted device

``dumpe2fs /dev/sdc1 | less`` - to see the superblock metadata
- ``dumpe2fs /dev/sdc1 | grep -i superblock``  - lists primary and backups superblock locations

Inode = index node
- many filesystems use inode
- each object including a file has its own inode
- contains metadata about the file or directory or object such as the stuff when we run ``ls -l`` except the filename and content
- number of inodes have a direct bearing on the number of files we can create
- ``ls -i`` to see the inode number of the file

Labels
- device file name can change eg /dev/sdc1 so we uuid
- uuid are hard to read, so can use labels
- ``blkid`` to see label 
- ``e2label /dev/sdc1 "willy label"`` to set label
  - ``blkid`` to check
- ``mke2fs -t ext4 -L "willy label" /dev/sdc1`` - we can create a label when we make the filesystem as well using ``L`` option

### 2.6 Removing and repairing filesystems



