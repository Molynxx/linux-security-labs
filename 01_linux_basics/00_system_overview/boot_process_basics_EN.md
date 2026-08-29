# boot_process_basics

## Lab goal
Understand how the system boot process works and how this knowledge is relevant to the detection of backdoors which load before the system starts. 

## What is boot process
This is a sequence of events from the moment of switching on the computer to the moment when it shows the login panel. The knowledge of this process is important to SOC, because tools such as `ps` and `ls` don't see backdoors which start before the system starts. 

## Boot process stage
- `BIOS/UEFI` - in this stage the correct operation of the hardware is checked (POST - Power-On Self Test), if everything is correct, motherboard doesn't give a warning about an error -> it searches for the bootloader on the drive, 
- `Bootloader (GRUB)` - loads the kernel and initramfs, shows a menu for choosing the system, 
- `Kernel` - the kernel takes control, initializes the hardware and mounts initramfs, 
- `Initramfs` - a temporary file system in memory, the drivers are loaded and the correct file system is mounted, 
- `Init systemd` - it starts the first process (PID 1), which starts services (network, SSH, logs).

## The most threats for SOC and detect methods
- `GRUB`:
	- threat: an attacker has access to the root account or direct access to the console and edits `/etc/default/grub`, adds `init=/bin/bash` to line `GRUB_CMDLINE_LINUX_DEFAULT`. Next starts `update-grub` (Debian/Ubuntu) and restarts the system. As a result, after reset the system doesn't start `init/systemd` but straight away the bash shell as root, without logging in and without passwords. Practically this attack is effective only with physical access or through remote management (iLO/iDRAC). Ordinary remote connection (e.g. SSH) won't be possible, because the system doesn't start the network - however, iLO/iDRAC work independently of the system. 
	- how to detect: check `/etc/default/grub` and `/boot/grub/grub.conf` if it does not contain the entry `init=/bin/bash`. You can use the command: `cat /etc/default/grub | grep "init="`.
	- how to repair:
		- delete `init=/bin/bash` from `/etc/default.grub`, 
		- run `sudo update-grub` (Debian/Ubuntu), 
		- reboot the system, change passwords and check for other backdoors - attackers often leave additional backdoors in the system. 
- `initramfs`:
	- threat: initramfs is a temporary filesystem, which is loaded into RAM with the kernel. It contains the necessary scripts and drivers to mount the correct file system from the drive. Initramfs is not a single file, it is an archive containing all directories with scripts and programs. It is main startup script named `init` and runs first. Attackers (who have access to root) can:
		- unpack the initramfs archive, 
		- read the contents of the unpacked files, 
		- add their own malicious script. 
		- repack and replace the image. 
	This code runs before the system starts and is invisible to `ps`, `ls` and `netstat`, because the system is not running yet. It makes a connection to an address specified by the attacker, therefore no incoming connections that could rise suspicion are visible either. It does not require physical access. 
	- how to detect: the best method is to use the command `lsinitramfs` which shows the list of files located in initramfs without unpacking it. Sample commands: 
		- `lsinitramfs /boot/initrd.img-$(uname -r) | head -30` - check if initramfs contains unknown files, 
		- `lsinitramfs /boot/initrd.img-$(uname -r) | grep -E "\.sh|\.so"` - look for scripts and libraries. 
	- how to repair: 
		- the simplest method is reinstalling the kernel package using the command `sudo apt install --reinstall linux-image-$(uname -r)` (Debian/Ubuntu), 
		- you can fix it manually too -> unpack the archive, delete suspicious files and pack it again. 
- `kernel`:
	- threat: kernel replacement (kernel rootkit). An attacker with root access can:
		- compile their own system kernel or download a ready-made one, 
		- replace `/boot/vmlinuz-$(uname -r)` with their own file, 
		- restart the system,
		A kernel-level rootkit is the program which: 
			- injects itself into the system kernel, 
			- hooks system calls (syscalls), 
			- filters responses - deletes those that match its configuration e.g PID, file name, port, IP. 
			It is important to remember that programs such as `ps`, `ls`, or `netstat` run in user space. To obtain information about processes, network connections or files, these programs ask the system kernel through system calls. Since the rootkit runs inside the kernel, it can intercept and modify answers. Therefore: 
				- when `ps` asks about processes, rootkit deletes from the responses processes whose PID or name are in its configuration, 
				- when `ls` asks about files, rootkit removes the responses that it has been programmed to hide, 
				- the same happens for other system calls such as `netstat`, `who`, `lsmod`. 
				Even special tools, created to detect rootkits such as `rkhunter` or `chkrootkit` might not work because they rely on the same system calls.
	The threat is so dangerous because the kernel replacement process does not require physical access, it works through SSH as well - the only condition is remote access to root. 
	- how to detect: 
		- a good practice is to save the checksum immediately after system installation, it makes it possible to check at any moment whether they match the original values. If the result is `OK` the kernel was not replaced, `FAIL` result means the kernel has been replaced. 
			- `md5sum /boot/vmlinuz-$(uname -r) > /root/kernel_md5.txt` - save the checksum, after system installation, 
			- `md5sum -c /root/kernel_md5.txt` - compare the current with the initial one.
		- another method is comparison with the package (Debian/Ubuntu): 
			- `sudo debsums$(dpkg -S /bootvmlinuz-$(uname -r) | cut -d: -f1)` - verify the package checksum. The `debsums` command compares the checksum of the kernel file with the correct value saved in the database and returns `OK` if the checksums match, or `FAIL` if the checksums don't match, 
			- you can check the modification date of the kernel file too: `ls -la /boot/vmlinuz-$(uname -r)`, if the date is different from the installation date - the kernel has been changed. 
	- how to repair: 
		- `sudo apt install --reinstall linux-image-$(uname -r)`  - reinstall the kernel package (Debian/Ubuntu), 
		- after re-installation you need to restart the system. 
		However, this is not always able to remove a kernel rootkit, there are situations where the only working method is reinstalling the entire system or restoring from a backup. 
- `systemd`: 
	- threat: an attacker can add a service, which runs the script e.g. a reverse shell. The service starts automatically every time the system boots up and runs as root. If the service goes down systemd restarts it. 
	- how to detect: 
		- `systemctl list-unit-files --type=service --state=enabled` - list of all services launched when the system starts, 
		- `systemctl list-units --type=service --state=running` - list of active services,
		- `systemctl cat service_name.service` - reads service details, 
		- `systemctl list-unit-files --type=service --state=enabled | grep -E "backup|update|fix|temp|cache"` - check if there are services with strange names (e.g. backup.service, system-update.service, network-fix.service, cron-helper.service), 
		- if the service looks suspicious you should check:
			- `systemctl cat name.service | grep ExecStart` - where the script is located. If ExecStart points to a directory `/tmp`, `/opt`, `/usr/local/bin` or `/lib/systemd/` this is a red flag, 
			- `cat /usr/local/bin/backdoor.sh` - if the script exists and what it contains. It is important to note if the script contains `nc -e /bin/bash`, `bash -i >& /dev/tcp/`, `python -c import socket...` - this means reverse shell, 
			- `ls -la /usr/local/bin/backdoor.sh` - who is the script owner. If the script has modification date after the attack or the owner is a user other than root (e.g. www-data) this is alarming, 
	- how to repair (remediation): 
		- `systemctl stop service_name` - stop the service, 
		- `systemctl disable service_name` - disable the service so it doesn't start at boot. 
		- `rm /etc/systemd/system/service_name` - remove the service file, 
		- `rm /usr/local/bin/backdoor.sh` - remove script, 
		- `systemctl daemon-reload` - reload systemd (refresh the service list), 
		- check if there are other backdoors in the system (cron, other services, GRUB, initramfs). 

## Safety conclusions 
All of the attacks described above require root privileges to be completed successfully. So if any of them was successful it means that the attacker obtained access to root, therefore besides the above solutions, more steps should be taken to improve root security. Without this procedure all of repair actions will not be effective, because most important is cut the attacker off from the possibilities, which give access to root. The priority is:
- root account protection - very strong password, disabled SSH login (or allow logging only with a key), log monitoring,
- trusted SSH keys monitoring - `/root/.ssh/authorized_keys` and `/home/*/.ssh/authorized_keys`, 
- regular root password changes, especially after suspicious events, 
- root access audit - `sudo -l`, `journalctl _UID=0`, `grep "COMMAND" /var/log/auth.log`. 


