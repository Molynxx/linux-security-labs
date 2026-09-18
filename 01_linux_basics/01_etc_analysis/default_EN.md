# /etc/default

## Goal
Review the directory containing default settings for Linux system commands. 

## What is `/etc/default`
This is a directory with files containing default settings for various tools and services. They are read during system boot, unless other information is explicitly provided. 

## Most important files in the directory

### `/etc/default/useradd` 
This is a file, which sets default values for `useradd` command. 
- essential file parameters:
	- `HOME` - determines home directory, where home directories are created, 
	- `SHELL` - default shell for new users (`/bin/bash`), 
	- `INACTIVE` - how many days after the password has expired an account is blocked (`-1` - never),
- threats:
	- `SHELL=/bin/false` or `/sbin/nologin` set in the `SHELL` parameter causes new users to be unable to log in, which may be deliberate (e.g. service accounts) or an administration mistake, 
	- `SHELL=/home/attacker/shell` - an attacker can take over a new account (applying their own shell),
	- `HOME=/tmp` - home directories in `/tmp` mean that anyone can read other users' data, 
	- `INACTIVE=-1` - the account will never be blocked after the password has expired. It is good practice to set `INACTIVE=0` value which will ensure that the account is blocked immediately after the password has expired.
- detection:
	- `grep -E "HOME|SHELL|INACTIVE|EXPIRE" /etc/default/useradd` - checking for incorrect parameter settings, 
- repair:
	- `HOME=/home` for the `HOME` parameter, 
	- `SHELL=/bin/bash` for the `SHELL` parameter, 
	- `INACTIVE=0` for the `INACTIVE` parameter, 
	- `EXPIRE=` for the `EXPIRE` parameter. 

### /etc/default/grub
This is GRUB bootloader configuration, it influences how the system starts up.
- essential parameters:
	- `GRUB_CMDLINE_LINUX_DEFAULT` - parameters passed to the kernel during a normal boot, 
	- `GRUB_CMDLINE_LINUX` - parameters passed to the kernel in emergency mode, 
	- `GRUB_TIMEOUT` - time (in seconds) to choose a system in GRUB menu. 
- threats:
	- added entry `init=/bin/bash` or `single` or `1` (single user mode) in `GRUB_CMDLINE_LINUX_DEFAULT`. These are parameters that start the system in a single user mode, which also gives access to the root shell. In this mode the system starts up without network services and minimal environment, but it gives an immediate root shell (without asking for the password). If an attacker has physical access this allows them to bypass the entire authentication mechanism using entry `single` or `1`. After restart the system starts up straight away to the root's shell (without login and password). 
- detecting:
	- `cat /etc/default/grub | grep "init="`, 
	- `grep "init=" /boot/grub/grub.cfg`.   
The commands mentioned above allow checking the value entered in `init` parameter. It should be checked if this parameter has a suspicious value (e.g. `/bin/bash`). In a correct configuration `init` parameter doesn't exist in `GRUB_CMDLINE_LINUX_DEFAULT`. 
- repair:
	- remove/change the suspicious parameter from `/etc/default/grub`, 
	- run the `sudo update-grub`, 
	- restart the system.   
Important: the GRUB modification requires root access or physical access to the console. This is not a vector for a remote attacker without privileges. 

### /etc/default/cron
This file sets environment variables for cron jobs. 
- threats: 
	- low risk, an attacker could set the `PATH` variable or `LD_PRELOAD` variable for cron, but it requires the root access and in that case he already has full root access anyway. Setting `LD_PRELOAD` in this file allows injecting a library into all cron jobs launched as root. This means that each planned script (e.g. backup, clearing logs) will be performed with the injected library which e.g. sends the data, creates backdoors or changes command results - all without visible changes to the cron scripts themselves. Although it requires root access, it is an effective method of maintaining access (persistence) - it survives system restarts, is not visible in `crontab -l` and the injected library is executed every time a scheduled cron job runs, which makes it difficult to detect during a standard audit. 
- detecting:
	- `/etc/default/cron` - the environment variable settings should be checked. (Correct environment variable settings are described in more detail in the file `05_pam_basics/security/pam_env_EN.md`). 
- repair:
	- restore the correct environment variable values. 

### /etc/default/locale
This file sets default language variables for the system. (Described in more detail in the file `05_pam_basics/security/pam_env_EN.md`). 
- threats:
	- changing `LANG` or `LC_ALL` may disrupt the operation of scripts. Changing `LANG` to a different (e.g. from pl_PL.UTF-8 to en_US.UTF-8) may change date format in logs, which makes it difficult to parse them automatically using SIEM or analytics scripts. An attacker may deliberately set an unusual locale to delay detection or cause confusion during analysis. 
- detecting: 
	- `cat /etc/default/locale` - it should be checked `LANG` and  `LC_ALL` values. 
- repair:
	- restore correct parameter values. 

### /etc/default/ssh
This file is usually empty in contemporary systems, SSH has its own configuration in the `/etc/ssh/sshd_config` file. 

## Safety conclusions
`/etc/default` is not the first place where an attacker hits because most of the files located here require root access for modification. If the attacker has root access already, they can cause far more damage than simply modifying files in this directory. However, it's worth keeping in mind that such a risk exists.