# System Users Basics

## The goal of the laboratory
The goal of this laboratory was to understand different types of Linux system users to improve log analysis (e.g. /var/log/) and identify potential security risk related to unauthorized access.

## Steps taken
- review of the /etc/passwd file to:
	- identify types of users,
	- understand the meaning of fields such as username, UID, home directory and shell form a security perspective,
- analysis of the 'last' commend to check the history of successful logins.

## Observations 
- There are tree types of users in the system:
	- root - an administrative user with the full privileges,
	- system users - accounts used by services and daemons,
	- interactive users (people) - accounts intended for user logins.
- Interactive users:
	- usually have a home directory in /home,
	- have an assigned shell (e.g. /bin/bash, /bin/zsh).
- System users:
	- often do not have a home directory or use technical directory (e.g. /var/lin/...),
	- system users should not be allowed to log in interactively.
- root user has a system shell, but its use should be restricted and controlled.

## Security conclusions
- A potential threat is a system user possessing an interactive shell (/bin/bash, /bin/zsh), because this enable log in,
- system users should have ability to log in blocked (using nologin or false),
- interactive users constitute a natural attack vector (e.g. account takeover, brute force attacks),
- upon detecting a suspicious user, it is necessary to:
	- determine when the account was created (if available e.g. backups, auditd, provisioning logs),
	- verify whether the account was reported and authorized,
	- check whether logins occurred locally or remotely (e.g via SSH),
	- analyze the logins history.
- For analysis:
	- the 'last' command is used check successful logins,
	- the 'lastb' command is used to check failed logins, if the tool is available in the distribution.

## Monitoring of Interactive Accounts (SOC/IR)
Interactive accounts are one of the major attack vectors in Linux system. From the SOC/IR perspective it is crucial not only to track the existence accounts, but also to continuously monitor their activity. In production environments, SOC teams create alerts based on:
- logins at unusual hours, 
- logins from new accounts or unusual IP addresses,
- logins to accounts that are typically not used,
- sequences of multiple failed logins attempts followed by a successful login.
For local event analysis, the following tools are commonly used:
- last - history of successful logins,
- jouralctl -u ssh --since "time" or analysis /var/log./auth.log,
- correlation of timestamp, username and login source.

## Limitations of the lab
- In my Kali Linux installation, the file /var/log/btmp exists, however the available 'last' tool is unable to read its contents due to format error. The 'lastb' command is not available as a separate tool. This limits the ability to analyze failed logins attempts using standard utilities.
- The system was freshly installed and did not contain data simulating a real security incident. 