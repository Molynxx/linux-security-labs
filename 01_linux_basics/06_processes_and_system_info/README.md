# 06_processes_and_system_info

## Cel
Zrozumienie, jak monitorować procesy, analizować katalog `/proc`, sprawdzać podstawowe informacje o systemie oraz zarządzać zmiennymi środowiskowymi. 

## Zakres
- `ps_top_htop` - monitorowanie procesów: `ps` (snapshot), `top` / `htop` (na żywo), analiza podejrzanych procesów (lokalizacja, nazwa, użytkownik, sieć, CPU), case study z fałszywym `kworker`,
- `proc_filesystem` - wirtualny system plików `/proc`, najważniejsze wpisy (`cmdline`, `exe`, `fd`, `environ`, `status`, `maps`, `stack`), oznaczenia (`txt`, `(deleted)`, `REG`), praktyczne polecenia,
- `system_identity` - podstawowe informacje o systemie: jądro (`unam -a`), restarty (`uptime`, `last reboot`), nazwa hosta, adres IP, dystrybucja (`/etc/os-release`). Zagrożenie i weryfikacja, 
- `environment_variables` - zmienne środowiskowe: sprawdzania (`env`, `set`, `echo`), ustawienia tymczasowe (`export`) i trwałe (`.bashrc`, `.profile`, `/etc/environmnet`, `pam_env.conf`), usuwanie (`unset`). Odesłanie do szczegółowej analizy zagrożeń w `05_pam_basics/security/pam_env_PL.md`.

## Dlaczego to ważne (SOC/IR)
- `procesy`- szybkie wykrywanie backdoorów, minerów, reverse shell, 
- `/proc` - weryfikacja prawdziwych ścieżek i argumentów procesów (nawet jeśli `ps` kłamie),
- `tożsamość systemu` - podstawa do zrozumienia ataków przez `LD_PRELOAD`, `PATH`, `IFS`.

## UWAGA
Szczegółowa analiza zagrożeń związanych ze zmiennymi środowiskowymi znajduje się w `05_pam_basics/security/pam_env_PL.md`.