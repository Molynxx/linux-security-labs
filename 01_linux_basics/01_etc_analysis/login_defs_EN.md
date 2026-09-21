# /etc/login.defs

## Goal
Understand what meaning the `/etc/login.defs` configuration file has for system security. 

## What is /etc/login.defs
This is the configuration file for the shadow-utils tools, such as `useradd`, `usermod`, `userdel`, `passwd`, `login`. Some PAM modules also use this file:
- `pam_unix.so` - determines the hash algorithm defined in the `ENCRYPT_METHOD` parameter and the complexity od hashing from the `SHA_CRYPT_MAX_ROUNDS` parameter, 
- `pam_faildeelay.so` - reads the `FAIL_DELAY` parameter value, which determines deal after failed login attempt, 
- `pam_umask.so` - reads the default privileges from the `UMASK` parameter for new files.   
Other parameters don't apply to PAM. It is necessary to remember that `login.defs` is a configuration file and shouldn't be confused with `login`, which is a program and has its own configuration in PAM. 

## Essential parameters for SOC
- password policy:
	- `PASS_MAX_DAYS` - determines after how many days the password must be changed, large values mean the risk that a compromised password works for a very long time, 
	- `PASS_MIN_DAYS` - this is the minimal interval between password changes which prevents an immediate password change back to the old one, 
	- `PASS_WARN_AGE` - determines how many days before the password expires the system warns, thank to this, users are informed about necessary password change.   
	It is important to distinguish the purpose of the file `/etc/login.defs` from the operation of the configuration file for the `pam_pwquality.so` module, the `/etc/login.defs` file defines the password aging policy, so when the password expires and how often it may be changed. While the `/etc.security/pwquality.conf` file defines the password quality policy, so how long the password should be, how many character classes, etc. 
- users' accounts:
	- `UID_MIN` / `UID_MAX` - UID range for ordinary users, in most contemporary distributions it is set to 1000 and more to clearly separate UID for system accounts and interactive accounts which makes monitoring and auditing easier, 
	- `SYS_UIID_MIN` / `SYS_UID_MAX` - analogously to the above, this is the UID range for system accounts, 
- `su` restriction:
	- `SU_WHEEL_ONLY` - set to 'yes' means that only the wheel group can use `su` , this is an essential layer of security against `su` attempts by non-privileged users. NOTE: in some distributions, `su` restriction to the wheel group is performed by the `pam_eheel.so` PAM module in the `/etc/pam.d/su` file, 
- privileges:
	- `UMASK` - default privileges for newly created files. `UMASK` works by removing privileges. It is important, as thee entry 077 strips all privileges for the group and others, so the owner has access. (NOTE: root always has access, regardless of privileges). The recommended value is 027 or 077 to protect users' data from others' access. 
- home directory:
	- `CREATE_HOME` - applies to the `useradd` command, if it is set to 'yes' the command will create a home directory by default when user adding, 
- password hashing:
	- `ENCRYPT_METHOD` - determines the hashing algorithm (SHA-512, YESCRYPT, etc.). This parameter is of great importance for security, because a weak hashing algorithm increases the risk of password cracking,
	- `SHA_CRYPT_MAX_ROUNDS` - determines the complexity  of hashing, a low value means faster hashing, but weaker hashing ass well. How is this: the system takes the password, hashes it and next takes the result and hashes it again, and repeats it thousands of times. Thanks to this cracking the password by an attacker takes years and it is impossible to crack it in a few second, minutes or hours. For SHA-512, the default is 5000 rounds.
- delay on error:
	- `FAIL_DELAY` - the delay in seconds occurring after typing an incorrect password, makes brute force attacks more difficult. This parameter is used by the `pam_faildelay.so` PAM module. 

## Threats
An attacker who has root access has a few possibilities, they may:
- extend the time in `PASS_MAX_DAYS` to delay enforcing password changes for all users, 
- change `UID_MIN` to the system range value so their account looks like a service and doesn't rise suspicion at first glance - the change applies to newly created accounts only,
- turn off the `SU_WHEEL_ONLY` and add their own account to the wheel group top be able to use `su`, 
- change `WNCRYPT_METHOD` to a weak hashing algorithm (e.g. DES) to easily crack passwords, 
- set `FAILD_DELAY=0` to speed up brute force attacks. 

## Security conclusions
Monitoring the essential parameters in the `/etc/login.defs` file is very important for unauthorized changes. For this purpose, the following command are useful: 
- `grep -E "PASS_MAX_DAYS|PASS_MIN_DAYS|PASS_WARN_AGE" /etc/login.defs` - checking the password aging policy,
- `grep -E "UID_MIN|UID_MAX|STS_UID_MIN|SYS_UID_MAX" /etc/login.defs` - checking UID ranges, 
- `grep -E SU_WHEEL_ONLY /etc/login.defs` - checking `su` restrictions, 
- `grep -E "UMASK|ENCRYPT_METHOD|FAIL_DELAY" /etc/login.defs` - checking privileges, the hashing algorithm, and the delay, 
- `sudo find /etc/login.defs -type f -mtime -1 -ls` - checking if the file hasn't been modified in last days, 
- changes in `login.defs` file don't affect already existing accounts, so e.g. UID_MIN doesn't affect already existing users, 
- some changes (e.g `UMASK`) in 'login.defs` file require re-login, others (e.g. `ENCRYPT_METHOD`) are read during creating new user. 

