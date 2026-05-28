# Permissions and ownership

## Cel 
Zrozumienie systemu uprawnień w Linuxie, zarządzanie właścicielami plików i katalogów oraz praktyczne przypadki błędów konfiguracyjnych.

## Zakres
- `chmod_chown` - podstawy uprawnień (`rwx`, notacja ósemkowa), zmiana uprawnień (`chmod`) i właścicieli (`chown`, `chgrp`), zagrożenia i monitoring, 
- `umask` - domyślne ustawienia dla nowo tworzonych plików i katalogów, gdzie ustawić, dobre praktyki, 
- `suid_sgid_sticky_bit` - bity specjalne: `SUID` (uruchom jako właściciel), `SGID` (dziedziczenie grupy), `sticky bit` (tylko właściciel usuwa) - działanie, zagrożenia, detekcja,
- `file_permission_cases` - praktyczne case study: `/tmp`, `/etc/shadow`, `SUID` na vim, `umask 000`, `SGID` na katalogu współdzielonym

## Dlaczego to ważne (SOC/IR)  
- `nieprawidłowe uprawnienia` to częsty wektor eskalacji uprawnień i dostępu do poufnych danych, 
- `bity specjalne`  (`SUID`, `SGID`, `sticky bit`) - jeśli źle skonfigurowane, mogą umożliwić przejęcie kontroli nad systemem, 
- `umask` - zbyt niska maska powoduje, że pliki są dostępne dla nieuprawnionych użytkowników, 
- `monitoring` - regularne audytowanie uprawnień (`find`, `ls -l`). 

## UWAGA 
Szczegółowa analiza uprawnień w kontekście PAM i środowiska znajduje się w:
- `05_pam_basics/security/` - `limits.conf`, `pam_env.conf`,
- `02_users_and_groups/` - `passwd`, `shadow`, `group`.