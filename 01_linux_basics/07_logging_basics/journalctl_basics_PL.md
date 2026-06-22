# journalctl_basics

## Cel
Zrozumienie różnicy pomiędzy `journalctl` a `auth.log` i `syslog` oraz jak czytać wpisy w nim zawarte. 

## Czym jest journalctl
To narzędzie służące do przeglądania logów zbieranych przez `systemd-journald` - centralny system logowania w nowoczesnych dystrybucjach Linuxa. Logi te są przechowywane w formacie binarnym w pamięci RAM (w pliku `/run/log/journal/`), a nie w plikach tekstowych jak `auth.log` czy `syslog`. Plik konfiguracyjny `journalctl` znajduje się w pliku `/etc/systemd/journald.conf`, opcja `storage` określa czy logi są zapisywane na dysku. Domyślnie `storage` jest ustawione na `auto` co nie gwarantuje, że logi zostaną zapisane, to zależy od tego czy istnieje katalog `/var/log/journal/`. Najbezpieczniejszym wyborem jest zmiana tej opcji na wartość `persistent` - to wymusza zapisanie logów, a jeśli katalog `/var/log/journal/` nie istnieje, `systemd` go utworzy automatycznie. Kolejne kroki konieczne do włączenia trwałych logów `journalctl`:
- edycja pliku konfiguracyjnego jako `root` za pomocą polecenia: `sudo systemctl edit --full --force systemd-journald`, ustawiając opcję: `Storage=persistent`,
- zrestartowanie usługi, aby zmiany zaczęły obowiązywać: `sudo systemctl restart systemd-journald`,
- przeniesienie logów z pamięci na dysk za pomocą polecenia: `sudo journalctl --flush`. 

## Podstawowe opcje journalctl
- `journalctl` - wyświetla wszystkie logi od najstarszych, 
- `journalctl -r` - logi od najnowszych do najstarszych, 
- `journalctl -f` - monitorowanie na żywo (tail -f),
- `journalctl -u nazwa.service` - logi tylko dla konkretnej usługi, np. sshd, cron,
- `journalctl -p err` - logi o priorytecie `err` (błędy), można też stosować warning, info, debug, 
- `journalctl --since "1 hour ago"` - logi z ostatniej godziny, 
- `journalctl --since "2024-06-07 10:00:00" --until "2025-06-07 11:00:00"` - logi z określonego przedziału czasowego, 
- `journalctl --list-boots` - lista wszystkich bootów (uruchomień systemu),
- `journalctl -b -1` - logi z poprzedniego boota (-2 - sprzed dwóch, itp.),
- `journalctl -u sshd -b -1` logi SSH z poprzedniego boota, 
- `journalctl -k` - logi jądra (kernel),
- `journalctl -o json` - wyjście w formacie JSON (przydatne do SIEM),
- `journalctl --disk-usage` - pokazuje, ile miejsca zajmują logi.

## Filtrowanie
- `sudo journalctl -u sshd | grep "Failed password"` - nieudane logowania SSH,
- `sudo journalctl -u sshd | grep "Accepted"` - logowania roota, 
- `sudo journalctl | grep -E "/tmp|/home"` - podejrzane lokalizacje, 
- `sudo journalctl | grep -E "wget|curl|nc|python -c"` - podejrzane polecenia, 
- `sudo journalctl -k | grep -E "panic|soft lockup|out of memory"` - błędy jądra, 
- `sudo journalctl | grep -E "Started.*\.service" | grep -v "Started Session"` - nowe usługi.

## Przykłady 
- `sudo journalctl -u sshd -f` - podgląd logów SSH na żywo, 
- `sudo journalctl -p err --since "30 minutes ago"` - błędy z ostatnich 30 minut, 
- `sudo journalctl -u cron --since "2025-06-07 02:00:00" --until "2025-06-07 03:00:00"` = logi z zakresu czasu dla usługi cron, 
- `sudo journalctl -b -1` - logi z poprzedniego boota, 
- `sudo journalctl -k -b -1` - logi jądra z poprzedniego boota, 
- `sudo journalctl -u sshd -n 20` - ostatnie 20 logów SSH. 

## Wnioski bezpieczeństwa
`journalctl` to potężne narzędzie do analizy na żywo oraz filtrowania, lecz nie może całkowicie zastąpić `auth.log` i `syslog`, szczególnie jeśli nie został skonfigurowany `Storage=persistent`. W wielu środowiskach produkcyjnych logi są przesyłane do SIEM, który zbiera je z plików tekstowych. Należy pamiętać:
- Persistent wymaga konfiguracji - domyślne logi nie są trwałe, znikają po restarcie, 
- Logi są przechowywane w formacie binarnym, nie ma możliwości użycia `grep` bezpośrednio na plikach - wymagany jest `journalctl`,
- wiele poleceń wymaga uprawnień roota.  

## Case study
Serwer WWW (nginx) działa wolno, kilkukrotnie przestał odpowiadać w ciągu ostatnich 2 godzin. Fragment logów `journalctl`:
```
Jun 8 15:00:01 serwer nginx[1234]: 2025/06/08 15:00:01 [error] 1234#1234: *1 connect() failed (111:Connection refused) while connecting to upstream  
Jun 8 15:00:05 serwer nginx[1234]: 2025/06/08 15:00:05 [error] 1234#1234: *2 connect() failed (111:Connection refused) while connecting to upstream  
Jun 8 15:01:00 serwer systemd[1]: Started php-fpm.service  
Jun 8 15:02:00 serwer nginx[1234]: 2025/06/08 15:02:00 [error] 1234#1234: *3 FastCGI sent in stderr: "Primary script unknown" while reading response header from upstream  
Jun 8 15:05:00 serwer sudo[5678]:  jan : TTY=pts/0 ; PWD=/var/www/html ; USER=root ; COMMAND=/bin/ls -la  
Jun 8 15:06:00 serwer sudo[5678]:  jan : TTY=pts/0 ; PWD=/var/www/html ; USER=root ; COMMAND=/bin/chmod 777 /var/www/html/uploads  
Jun 8 15:10:00 serwer sshd[7890]: Accepted password for root from 192.168.1.100 port 54321 ssh2   
Jun 8 15:15:00 serwer kernel [12345]: Out of memory: Kill process 9876 (php-fpm) score 80   
Jun 8 15:16:00 serwer systemd[1]: Stopped php-fpm.service  
Jun 8 15:17:00 serwer systemd[1]: Started php-fpm.service  
```
Analiza:  
- należy sprawdzić dlaczego usługa php-fpm.service (usługa wykonująca kod PHP na zlecenie serwera) była wyłączona przed początkiem powyższych logów, 
- wpisy `connect() failed()` -  świadczą o problemie z połączeniem, z powodu wyłączonej usługi php-fpm, 
- dopiero po tych 2 błędach została włączona usługa php-fpm,
- użytkownik jan, przy  użyciu `sudo` wyświetla zawartość katalogu `/var/www/html`, a następnie zmienia uprawnienia dla podkatalogu `uploads`. To podejrzane działanie, umożliwiające każdemu otwieranie katalogu, zapisywanie w nim plików oraz ich odczyt, 
- kolejny wpis ukazuje logowanie z lokalnego IP, najprawdopodobniej to logowanie admina. Jednak lodowanie odbyło się przez SSH, bez względu na to czy to był admin czy nie, logowanie hasłem do roota przez SSH powinno być wyłączone (czerwona flaga) - logowanie powinno odbywać się za pomocą klucza,
- kolejne zdarzenie `Out of memory` ukazuje, że usługa php-fpm zajmuje dużo pamięci, procesy które uruchomiła zostały zabite przez system, co pośrednio zabija też usługę, 
- na końcu bieżących logów następuje zresetowanie usługi (prawdopodobnie przez admina, który dostrzegł, że usługa nie działa poprawnie), 
- to co tutaj zaszło wskazuje, że atakujący mający dostęp do konta jan posiadającego prawa do `sudo`, prawdopodobnie:
	- zatrzymał usługę (wcześniej, przed logami widocznymi powyżej) celem wgrania skryptów, usunięcia plików z katalogu `/var/www/html`, przez co nawet po uruchomieniu php-fpm strona nie działa, 
	- następnie przegląda katalog `/var/www/html` i ustawia pełne uprawnienia dla wszystkich dla podkatalogu `uploads`. 
	- webshell (skrypt PHP) umieszczony w `uploads/`, powoduje wyczerpanie pamięci, usługa zostaje zabita, strona znów nie działa.  

Zalecane działania:
- należy sprawdzić, kto i kiedy logował się na konto jan oraz jakie czynności wykonał, w czasie sesji, 
- należy zmienić hasło dla konta jan, 
- należy sprawdzić jakie zmiany zostały przeprowadzone w katalogu `/var/www/html`, cofnąć te zmiany oraz usunąć podejrzane pliki i katalogi, 
- sprawdzić pliki konfiguracyjne usługi znajdujące się w katalogu `/etc/php/*/fpm/pool.d/`, a następnie cofnąć wszystkie zmiany wprowadzone przez atakującego, 
- ponieważ skompromitowane konto jan posiadało dostęp do `sudo` należy przeprowadzić audyt systemu.
