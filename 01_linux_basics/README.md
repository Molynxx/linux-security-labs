# 01_linux_basics

Praktyczna dokumentacja nauki bezpieczeństwa systemu Linux z perspektywy SOC/Blue Team i Detection Engineering. Repozytorium powstało jako portfolio w procesie przebranżowienia z inżynierii budowlanej do cyberbezpieczeństwa.

## Cel
Zrozumienie mechanizmów działa systemy Linux, analizy logów, kontroli dostępu oraz wykrywania zagrożeń - wszystko w kontekście pracy w Security Operation Center. 

## Zakres
- `00_system_overview/` - struktura systemu plików, katalogi, proces botowania, 
- `01_etc_analysis/` - analiza plików konfiguracyjnych (`/etc/default`, `/etc/login.defs`, `/etc/security`),
- `02_users_and_gropus/` - użytkownicy i grupy: `/etc/passwd`, `/etc/group`. `/etc/shadow`, polecenia zarządzania, 
- `03_permissions_andownership/` - uprawnienia, chmod, chown, umask, SUID, SGID, sticky bit, 
- `04_sudo_and_priviilege_escalation_basics/` - sudo, sudoers, ryzyka eskalacji uprawnień, 
- `05_pam_basics/` - Pluggable Authentication Modules: Konfiguracja, typy, flagi, moduły, polityki, 
- `06_processes_and_system_info/` - procesy (`/proc`, `ps`, `top`, `htop`), zmienne środowiskowe, tożsamość systemu,
- `07_logging_basics/` - logi systemowe (`auth.log`, `syslog`, `journalctl`), filtrowanie grep,
- `08_tmp_and_files_locations/` - katalogi tymczasowe, wyszukiwanie plików (find, locate),
- `09_basic_security_checks/` - checklista audytu bezpieczeństwa Linux. 

## Metoda nauki
Każdy temat zawiera cześć teoretyczną i praktyczne przykłady. W kluczowych miejscach dodane są:
- `SOC perspective` - dlaczego to ważne dla analityka bezpieczeństwa, 
- `Case study` - analiza realistycznych scenariuszy (brute force, persistence, eskalacja). 

## Dla kogo
- dla osób przebranżawiających się do cyberbezpieczeństwa, 
- dla aspirujących analityków SOC i Detection Engineering, 
- dla każdego, kto chce zrozumieć Linuxa głębiej niż "wpisz to polecenie".

## Status 
Repozytorium `01_linux_basucs` - ukończone.   

Kolejne w przygotowaniu:
- `02_linux_advanced` - systemd, procesy, persistence, forensics, 
- `03_soc_linux_security_labs` - scenariusze detekcji i analizy. 

## Powiązane repozytoria
- `networking-for-soc` - podstawy sieci dla SOC (w budowie).