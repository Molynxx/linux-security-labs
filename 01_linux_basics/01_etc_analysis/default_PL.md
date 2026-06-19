# /etc/default

## Cel
Zapoznać się z katalogiem zawierającym ustawienia domyślne poleceń systemu Linux.

## Czym jest /etc/default
Jest to katalog z plikami zawierającymi domyślne ustawienia dla różnych narzędzi i usług. Są odczytywane podczas uruchamiania programu, jeśli nie podano jawnie innych informacji. 

## Najważniejsze pliki w katalogu

### /etc/default/useradd
To plik, który ustawia domyślne wartości dla polecenia `useradd`. 
- Kluczowe parametry pliku:
	- `HOME` - oznacza katalog, w którym tworzone są katalogi domowe (`/home`),
	- `SHELL` - domyślna powłoka dla nowych użytkowników (`/bin/bash`),
	- `INACTIVE` - po ilu dniach od wygaśnięcia hasła konto jest blokowane (`-1` - nigdy),
	- `EXPIRE` - data wygaśnięcia konta (pusta = brak).  
- zagrożenia:
	- `SHELL=/bin/false` lub `/sbin/nologin` ustawione w parametrze `SHELL` spowoduje, że nowi użytkownicy nie będą mogli się zalogować, co może być celowe (np. konta serwisowe) lub błędem administracyjnym,
	- `SHELL=/home/attacker/shell` - atakujący może przejąć nowe konto (wstawia swoją powłokę), 
	- `HOME=/tmp` - katalogi domowe w `/tmp` - każdy może czytać dane innych, 
	- `INACTIVE=-1` - konto nie zostanie zablokowane po wygaśnięciu hasła nigdy, dobrą praktyką jest ustawienie wartości `INACTIVE=0`, co zapewni, że konto zostanie zablokowane natychmiast po wygaśnięciu hasła. 
- wykrywanie:
	- `grep -E "HOME|SHELL|INACTIVE|EXPIRE" /etc/default/useradd` - sprawdzenie pod kątem nieprawidłowych ustawień parametrów. 
- naprawa: należy przywrócić poprawne wartości parametrów, pamiętając, że zmiana ma wpływ wyłącznie na nowo tworzone konta, nie dotyczy już istniejących:
	- `HOME=/home` dla parametru `HOME`,
	- `SHELL=/bin/bash` dla parametru `SHELL`,
	- `INACTIVE=0` dla parametru `INACTIVE`,
	- `EXPIRE=` dla parametru `EXPIRE`.

### /etc/default/grub
To konfiguracja bootloadera GRUB, wpływa na to, jak system się uruchamia.
- kluczowe parametry:
	- `GRUB_CMDLINE_LINUX_DEFAULT` - parametry przekazywane do jądra przy normalnym bootowaniu, 
	- `GRUB_CMDLINE_LINUX` - parametry przekazywane do jądra w trybie awaryjnym, 
	- `GRUB_TIMEOUT` - czas (w sekundach) na wybór systemu w menu GRUB.
- zagrożenia:
	- dodany wpis `init=/bin/bash` lub `single` lub `1` (tryb single user) w `GRUB_CMDLINE_LINUX_DEFAULT `. To są parametry uruchamiające system w trybie pojedynczego użytkownika, co również daje dostęp do powłoki roota. W tym trybie system uruchamia się bez usług sieciowych, z minimalnym środowiskiem, ale od razu daje powłokę roota (bez pytania o hasło). Jeśli atakujący ma fizyczny dostęp, pozwala mu na ominięcie całego mechanizmu uwierzytelniania, za pomocą dodania wpisu `single` lub `1`, 
	 - po restarcie system uruchamia się od razu do powłoki roota (bez logowania i hasła).
- wykrywanie:
	- `cat /etc/default/grub | grep "init="`, 
	- `grep "init=" /boot/grub/grub.cfg`,
Polecenia te pozwalają sprawdzić wartość wprowadzoną w parametrze `init`. Należy sprawdzić, czy ten parametr ma podejrzaną wartość (np. `/bin/bash`). W prawidłowej konfiguracji ten parametr nie występuje w `GRUB_CMDLINE_LINUX_DEFAULT`.
- naprawa:
	- usunąć/zmienić podejrzany parametr z `/etc/default/grub`,
	- uruchomić `sudo update-grub`,
	- zrestartować system.   
UWAGA: Modyfikacja GRUB wymaga dostępu root lub fizycznego dostępu do konsoli. To nie jest wektor dla zdalnego atakującego bez uprawnień. 

### /etc/default/cron
Plik ten ustawia zmienne środowiskowe dla zadań cron.
- zagrożenie: 
	- niskie, atakujący mógłby ustawić zmienną `PATH` lub `LD_PRELOAD` dla cron, ale wymaga to roota, a wtedy i tak już ma pełny dostęp do roota. Ustawienie `LD_PRELOAD` w tym pliku pozwala wstrzyknąć bibliotekę do wszystkich zadań cron uruchomionych jako root. Oznacza to, że każdy zaplanowany skrypt (np. backup, czyszczenie logów) zostanie wykonany z wstrzykniętą biblioteką, która może np. wysłać dane, tworzyć backdoory lub zmieniać wyniki poleceń - wszystko bez widocznych zmian w samych skryptach cron. Mimo że wymaga roota, jest to skuteczny mechanizm utrzymania dostępu (persistence) - przetrwa restarty systemu, nie jest widoczny w `crontab -l`, a wstrzyknięta biblioteka wykonuje się przy każdym zaplanowanym zadaniu cron, co utrudnia wykrycie podczas standardowego audytu. 
- wykrywanie:
	- `/etc/default/cron` - należy sprawdzić ustawienie zmiennych środowiskowych. (Poprawne ustawienie zmiennych środowiskowych jest szerzej opisane w pliku `05_pam_basics/security/pam_env_PL.md`).
- naprawa:
	- przywrócić poprawne wartości zmiennych środowiskowych.  

### /etc/default/locale
Plik ustawia domyślne zmienne językowe dla systemu. (Szerzej jest omówione w pliku 05_pam_basics/security/pam_env_PL.md)
- zagrożenia:
	- zmiana `LANG` lub `LC_ALL` może zburzyć działanie skryptów. Zmiana `LANG` na inną (np. z pl_PL.UTF-8 na en_US.UTF-8) może zmienić format daty w logach, co utrudnia ich automatyczne parsowanie przez SIEM lub skrypty analityczne. Atakujący może celowo ustawić nietypową lokalizację, by opóźnić detekcję lub wprowadzić zamieszanie podczas analizy. 
- wykrywanie:
	- `cat /etc/default/locale` - należy sprawdzić wartości dla `LANG` i `LC_ALL`. 
- naprawa:
	- przywrócić poprawne wartości parametrów. 

### /etc/default/ssh
To plik zwykle pusty w nowoczesnych systemach, SSH ma własną konfigurację w pliku `/etc/ssh/sshd_config`.

## Wnioski bezpieczeństwa 
`/etc/default/` nie jest pierwszym miejscem gdzie atakujący uderza, ponieważ większość plików znajdujących się tu wymaga roota do modyfikacji. Jeśli atakujący ma już root to może zrobić znacznie więcej szkód niż zmiana w plikach tego katalogu. Jednak warto wiedzieć, że takie zagrożenie istnieje. 