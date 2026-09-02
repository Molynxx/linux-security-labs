# 01_etc_analysis

### Goal
Understand what configuration files are located in the `/etc` directory and wy they are crucial to SOC. 

### Scope
- `login.defs_PL.md` - configuration files for shadow-utils, such as `useradd`, `usermod`, userdel, `passwd`, `login`. Some PAM modules use these settings, 
- `default_PL.md` - the directory contains files specifying default settings for `useradd`, `grub`, `cron`, `locale` and others,
- `security_PL.md` - this directory contains PAM modules configuration files (this file provides a general overview, a detailed description of these files can be found at `05_pam_basics/security`).

### Why it is important (SOC/IR)
- the directory `/etc` contains files specifying how the system works and what its security policies are,
- all changes in  `login.defs`, `default/useradd`, `security` might indicate potential privileges escalation or backdoor, 
- without knowledge of `/etc` it is difficult to perform log analysis, process analysis or conducting forensics.

## Polski / Polish

### Cel
Zrozumienie, jakie pliki konfiguracyjne znajdują się w katalogu `/etc` oraz dlaczego są kluczowe dla SOC.

### Zakres
- `login.defs_PL.md` - plik konfiguracyjny dla narzędzi shadow-utils, czyli useradd, usermod, userdel, passwd, login. Niektóre moduły PAM korzystają z tych ustawień,
- `default_PL.md` - katalog zawierający pliki określające domyślne ustawienia dla `useradd`, `grub`, `cron`, `locale` i innych,
- `security_PL.md` - to katalog zawierający pliki konfiguracyjne modułów PAM (w tym pliku zostały omówione ogólnie, szczegółowy opis tych plików znajduje się w lokalizacji repo `05_pam_basics/security`).

### Dlaczego to ważne (SOC/IR)
- katalog `/etc` zawiera pliki określające jak działa system i jakie są jego polityki bezpieczeństwa,
- wszelkie zmiany w `login.defs`, `default/useradd`, `security` mogą wskazywać na potencjalną eskalację uprawnień lub backdoor, 
- bez znajomości `/etc` trudno jest analizować logi, procesy lub przeprowadzać forensics.