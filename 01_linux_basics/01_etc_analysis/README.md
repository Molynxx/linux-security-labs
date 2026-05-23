# 01_etc_analysis

## Cel
Zrozumienie, jakie pliki konfiguracyjne znajdują się w katalogu `/etc` oraz dlaczego są kluczowe dla SOC.

## Zakres
- `login.defs_PL.md` - plik konfiguracyjny dla narzędzi shadow-utils, czyli useradd, usermod, userdel, passwd, login. Niektóre moduły PAM korzystają z tych ustawień,
- `default_PL.md` - katalog zawierający pliki określające domyślne ustawienia dla `useradd`, `grub`, `cron`, `locale` i innych,
- `security_PL.md` - to katalog zawierający pliki konfiguracyjne modułów PAM (w tym pliku zostały omówione ogólnie, szczegółowy opis tych plików znajduje się w lokalizacji repo `05_pam_basics/security`).

## Dlaczego to ważne (SOC/IR)
- katalog `/etc` zawiera pliki określające jak działa system i jakie są jego polityki bezpieczeństwa,
- wszelkie zmiany w `login.defs`, `default/useradd`, `security` mogą wskazywać na potencjalną eskalację uprawnień lub backdoor, 
- bez znajomości `/etc` trudno jest analizować logi, procesy lun przeprowadzać forensics.