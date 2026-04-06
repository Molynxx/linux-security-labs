# /etc/pam.d/sudo

## Cel laboratorium
Zapoznanie się z zawartością pliku /etc/pam.d/sudo oraz zrozumienie jaką rolę pełni plik w systemie Linux.

## Czym jest /etc/pam.d/sudo
Nie jest to plik konfiguracyjny aplikacji sudo (/etc/sudoers), jest to plik konfiguracyjny PAM dla tej aplikacji, a zapisy w nim nie określają kto ma uprawnienia sudo lecz sposób uwierzytelnienia użytkownika. 

## Flow
- użytkownik wpisuje sudo <polecenie>,
- sudo:
	- sprawdza sudoers (policy plugin),
	- jeśli trzeba -> uruchamia PAM,
	- PAM:
		- auth - czy hasło jest prawidłowe, 
		- account - czy konto jest niezablokowane lub niewygasłe,
		- session - przygotowanie środowiska, rejestracja sesji oraz obsługa mechanizmów audytu i limitów, 
	- wykonuje polecenie.
Należy zaznaczyć, że sesja jest ustawiana tylko dla jednego polecenia a po jego wykonaniu się zamyka. 

## Zawartość pliku 
W pliku tym zwykle znajdują się pliki globalne PAM:
- common-auth,
- common-account,
- common-session-noninteractive. 
W przypadku konieczności skonfigurowania pliku w sposób bardziej restrykcyjny można, poza globalnymi plikami, dodać poszczególne moduły typów. Typowe dodatkowe moduły:
- `pam_wheel.so` - z opcją `use_uid` - np. tylko grupa wheel może używać sudo,
- `pam_faillock.so` / `pam_tally2.so` - ochrona przed brute force,
- `pam_time.so` - ograniczenia czasowe, 
- `pam_access.so` - kontrola dostępu (IP, user, TTY),
- `pam_env.so` - konfiguracja środowiska specyficznie dla sudo. 
Flagi powinny być dobrane zgodnie z funkcją modułu. 
	- `required` - dla modułów krytycznych dla bezpieczeństwa systemu lub audytu (np. `pam_env.so`, `pam_loginuid.so`, `pam_limits.so`),
	- `optional` - dla pozostałych (np. `pam_systemd.so`, `pam_keyinit.so`, `pam_umask.so`),
Moduł `pam_systemd.so` jest istotny dla audytu, jednak często stosowany z flagą `optional`, aby uniknąć sytuacji, w której błąd systemd-logind uniemożliwi użycie sudo. 

## Wnioski bezpieczeństwa 
- bardzo istotne jest właściwe ustawienie flag, modułów i ich kolejności, 
- niewłaściwe flagi mogą prowadzić do utrudnienia audytu, obejścia uwierzytelnienia, limitów, a także umożliwić manipulację środowiskiem,
- kluczowa jest kompletność modułów, dlatego zawsze przed konfiguracją tego pliku należy zweryfikować czy konfiguracja modułów w pliku common-session-noninteractive jest zgodna z polityką bezpieczeństwa systemu. 
- opcja NOPASSWD (ustawiona w sudoers) pomija etap uwierzytelnienia (auth). Pomimo pominięcia etapu auth, nadal wykonywane są etapy account i session, co oznacza, że mechanizmy audytu nadal mogą działać.
- sudo zapamiętuje uwierzytelnienie użytkownika przez określony czas (timestamp_timeout), co pozwala na wykonywanie kolejnych poleceń bez ponownego podawania hasła. Mechanizm zwiększa wygodę użytkowania, jednak w określonych scenariuszach (np. współdzielone systemy, brak blokady sesji) może zwiększać ryzyko nieautoryzowanego użycia sudo. 
