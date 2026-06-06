# var_log_structure

## Cel
Zrozumienie, jakie pliki logów znajdują się w `/var/log/` i do czego służą. To miejsce gdzie SOC szuka śladów logowania, błędów, ataków. 

## Ważne pliki i katalogi w /var/log/
- `/var/log/auth.log` - przechowuje logi uwierzytelniania (logowania, sudo, PAM, SSH). To najważniejszy plik, tu znajdują się ślady logowań, nieudane próby, sudo, zmiany użytkowników,
- `/var/log/syslog` - przechowuje ogólne logi systemowe (jądro, usługi i aplikacje). Znajdują się tu błędy systemowe, problemy z usługami, podejrzane procesy,
- `/var/log/kern.log` - przechowuje logi jądra, problemy ze sprzętem, ataki na jądro, `kernel panic`, 
- `/var/log/dpkg.log` (Debian/Ubuntu) - przechowuje logi instalacji i aktualizacji pakietów. Sprawdzić tu można, czy ktoś nie zainstalował nieznanego oprogramowania,
- `/var/log/apt/` - przechowuje logi `apt`, podobnie jak dpkg.log,
- `/var/log/nginx/` - przechowuje logi serwera WWW (nginx). Znaleźć można ataki na aplikacje webowe czy skanowanie,
- `/var/log/apache2/` - j.w. logi WWW (apache2),
- `/var/log/mysql/` - przechowuje logi bazy danych MySQL, można tu znaleźć ataki na bazę danych,
- `/var/log/cron.log` - przechowuje logi zadań cron. Sprawdzić tu można, czy cron nie uruchamia podejrzanych skryptów,
- `/var/log/journal/` - przechowuje logi systemd (w formacie binarnym). Logi te czyta się przez `journalctl`. 

## Uprawnienia i dostęp
- większość plików w folderze `/var/log/` jest czytelna tylko dla roota (czasem również dla grupy admin),
- plik `auth.log` i jemu podobne jest czytelne wyłącznie dla roota, ponieważ zawierają dane uwierzytelniające,
- `lastlog`, `wtmp`, `btmp` - są czytane przez polecenia `last`, `lastlog`, `lastb`.

## Rotacja logów
Pliki logów są rotowane (archiwizowane, kompresowane, usuwane) przez `logrotate`. Przykład:
- `auth.log` - aktywny, bieżący log,
- `auth.log.1` - starszy, nieskompresowany,
- `auth.log.2.gz` - skompresowany, jeszcze starszy niż poprzedni,
- najstarsze są usuwane.  
Narzędzie `logrotate` jest uruchamiane codziennie przez `cron` (`/etc/cron.daily/logrotate`). Plik konfiguracyjny dla tego narzędzia znajduje się w `/etc/logrotate.d/` i można w nim skonfigurować częstotliwość rotacji, ilość zachowanych starych wersji logów, kompresji starych logów. 

## Monitoring
- podejrzenie ataku brute force - należy sprawdzić plik `auth.log`, za pomocą polecenia:
	- `grep "Failed password" /var/log/auth.log`,
- podejrzenie nieautoryzowanego dostępu - sprawdzić `auth.log`:
	- `grep "Accepted" /var/log/auth.log`,
- podejrzenie ataku na usługę (SSH, Apache) - należy sprawdzić `auth.log` oraz `/nginx/error.log`:
	- `grep "ssh" /var/log/auth.log`,
	- `grep "error" /var/log/nginx/error.log`,
- sprawdzenie, czy logi nie zostały wyczyszczone - należy szukać dziur czasowych, braku logów o określonej godzinie.

## Case study
Fragment `auth.log`:  
```
Jun 6 10:00:01 server sshd[1234]: Failed password for root from 45.33.22.11 port 54321 ssh2  
Jun 6 10:00:03 server sshd[1235]: Failed password for root from 45.33.22.11 port 54322 ssh2  
Jun 6 10:00:05 server sshd[1236]: Failed password for root from 45.33.22.11 port 54323 ssh2  
... (150 takich wpisów)  
Jun 6 10:05:01 server sshd[1500]: Accepted password for root from 192.168.1.100 port 54321 ssh2  
```
Analiza:  
- został przeprowadzony atak brute force przez ssh z IP 45.33.22.11,
- atak się nie powiódł, nie ma żadnego wpisu zawierającego Accepted password for root from 45.33.22.11,
- o 10:05:01 logowanie udane z lokalnego IP, najprawdopodobniej logowanie admina,
- nieudane próby wpisania hasła występowały w odstępie 2 sekund, w związku z tym:
	- nie został ustawiony poprawnie moduł `pam_faildelay.so`, który ma za zadnie wprowadzać opóźnienie po nieudanym logowaniu lub
	- czas opóźnienia w tym module jest zbyt krótki,
- 150 prób logowania świadczy o braku właściwej konfiguracji modułu `pam_faillock.so`. Moduł nie jest poprawnie ustawiony w PAM lub nie wykorzystuje wszystkich trybów, koniecznych do ograniczenia zbyt wielu nieudanych prób. Moduł ten działa w 3 trybach zależnie od konfiguracji w typie `auth`:
	- preauth - ma za zadanie sprawdzać licznik nieudanych prób, 
	- authfail - po nieudanym logowaniu inkrementuje licznik nieudanych prób, 
	- authsucc - resetuje licznik po udanym logowaniu. 
- plik konfiguracyjny modułu `pam_faillock.so`, może nie zawierać odpowiedniej wartości dla opcji `deny`.   
Zalecane działania:  
- należy sprawdzić i poprawnie ustawić moduł `pam_faillock.so` w typie `auth` - powinien zawierać wszystkie tryby do poprawnego działania i zliczania prób, 
- należy sprawdzić plik konfiguracyjny modułu `/etc/security/faillock.conf` czy poprawnie została przypisana wartość do opcji `deny`, 
- próba była nieudana więc nie ma konieczności zmiany hasła dla roota, jednak admin może zmienić hasło prewencyjnie, 
- należy zablokować IP, z którego nadszedł atak.