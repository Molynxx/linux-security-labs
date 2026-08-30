# Filesystem_structure 

## Lab goal
Understand what the Linux file system looks like. 

## What the FHS is
The file system on which Linux is based, is named FHS (File Hierarchy Standard). It is a standard which defines what each directory is used for. As a result independently of the Linux distribution (Kali, Ubuntu, Debian, RHEL, Arch) we know that:
- logs are in `/var/log`, 
- the system configuration is in `/etc`, 
- user's programs are in `/usr/bin`.   
All of these directories are in main system directory `root` (/). 

## Main directories in `/` (root)
- `/bin` - basics user's programs (`ls`, `cp`, `mv`, `cat`), 
- `/sbin` - programs for administrators (`fdisk`, `mount`, `reboot`), 
- `/etc` - system configuration and service configuration (`passwd`, `shadow`, `sudoers`), 
- `/var` - variable data - logs, mail, cache, 
- `/tmp` - temporary files (clear upon restart, if tmpfs exists), 
- `/home` - ordinary users home directories,
- `/root` - root home directory, 
- `/boot` - files necessary to system boot (kernel, GRUB), 
- `/dev` - device files (drivers, terminals),
- `/proc` - virtual file system in the memory (procfs) - processes and hardware information, 
- `/sys` - virtual file system in memory (sysfs) - interaction with kernel and devices, 
- `/usr` - programs, libraries, user's documentation, this is a secondary main program directory after `/bin`, 
- `/lib` - libraries shared for programs from `/bin` and `/sbin`, 
- `/opt` - additional packages (third-part software), 
- `/media` - automatically devices mount (pendrive, CD),
- `/srv` -  service's data (www, ftp),
- `/run` - program data, process data, from the moment the system starts, 
- `/var/tmp` - temporary files (e.g. application's), which must be saved between reboots, it is not cleared upon reboot as `/tmp`,
- `/lib64` - in some distributions the `/lib64` exists for 64-bit libraries.

## Notice 
In modern distributions `/bin`, `/sbin`, `/lib` are often symbolic links to the proper directories in `/usr` (e.g. `/bin` -> `/usr/bin`).  