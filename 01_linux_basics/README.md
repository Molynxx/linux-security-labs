# 01_linux_basics

Podstawy działania systemu Linux w kontekście bezpieczeństwa - od struktury po audyt i analizę logów.

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

## Status 
Repozytorium `01_linux_basucs` - ukończone.   
