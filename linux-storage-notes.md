# Linux storage notes

## 1 Disks and partitions

### 1.1 Intro to local storage

Local storage means any storage devices plugged directly into the server (called internal drive)
- typically disk drives, SSD flash drives, dvd drives, blu-ray drives etc
- drives are normally plugged using SATA, SAS or SCSI adapters to the back of the server
- opposite of local storage _networked storage_, NAS and SAN are two major forms of networked storage

### 1.2 Devices and device files

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


8:01