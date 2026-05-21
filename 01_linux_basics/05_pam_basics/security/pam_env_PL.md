# /etc/security/pam_env.conf

## Cel
Zapoznanie się z opcjami konfiguracyjnymi modułu `pam_env.so` znajdującymi się w pliku konfiguracyjnym `/etc/security/pam_env.conf`.

## Do czego służy moduł pam_env.so
Jest to moduł ustawiający zmienne środowiskowe na podstawie:
- `/etc/security/pam_env.conf`,
- `/etc/environment`.
- opcjonalnie w pliku użytkownika `~/.pam_environment`.

## Jaka jest różnica pomiędzy plikami `/etc/security/pam_env.conf` i `/etc/environment`
- `/etc/security/pam_env.conf` - to zaawansowany, warunkowy i elastyczny plik konfiguracji modułu `pam_env.so`. Oferuje on znacznie większe możliwości niż `/etc/environment` ale jednocześnie ma bardziej skomplikowaną składnię. Jest on bardzo przydatny gdy potrzeba logiki.
- `/etc/environment` - to plik służący do ustawiania prostych zmiennych środowiskowych w formacie KEY=VAL dla wszystkich użytkowników i wszystkich procesów. UWAGA: jeśli w konfiguracji PAM (`sshd`, `common-session`) jest włączone `user_readenv=1`, to zwykły użytkownik może ustawić własne zmienne środowiskowe, a to stwarza cały szereg zagrożeń w tym wstrzyknięcie biblioteki, podmianę PATH itp. Ustawienie opcji `user_readenv=0` to kompletne i całkowite wyłączenie możliwości użycia pliku `~/.pam_environment`.

## Dlaczego dwa pliki dla jednego modułu
Ponieważ moduł `pam_env.so` działa w kilku warstwach:
- Warstwa 1 -  `/etc/security/pam_env.conf`, 
- Warstwa 2 -  `/etc/environment`, 
- Warstwa 3 - opcjonalnie `~/.pam_environment` - domyślnie wyłączony w nowoczesnych systemach.

## Składnia `pam_env.conf`
Należy pamiętać, że w składni kluczowe są składniki takie jak DEFAULT i OVERRIDE:
- `DEFAULT` - ustawi zmienną jeśli nie została jeszcze ustawiona, jeśli istnieje zostawi ją bez zmian. 
- `OVERRIDE` - zawsze nadpisuje zmienną, nawet jeśli już została ustawiona. `OVERRIDE` ma najwyższy priorytet.   
Przykład ustawienie PATH:  
`PATH OVERRIDE=/usr/local/bin:/usr/bin:/bin:/opt/bin`  

## Ważne parametry `pam_env.conf` i zagrożenia z nimi związane
- `LD_PRELOAD`: 
	- co określa ta zmienna: to ścieżka wskazująca, która biblioteka ma być załadowana jako pierwsza (przed standardowymi).
	- zagrożenie: wstrzyknięcie biblioteki przez atakującego, dzięki czemu polecenie może wykonać zupełnie inne zadanie, np. wykonać kod atakującego. 
	- jak wykryć: 
		- sprawdzić miejsca gdzie ścieżka dla zmiennej mogła zostać ukryta, czyli:
			- `~/.bashrc`, 
			- `~/.profile`,
			- `/etc/security/pam_env.conf`.
	- jak naprawić: 
		- w shell poleceniem `export LD_PRELOAD=...` lub w pliku `pam_env.conf`, pusta wartość ustawi standardowe biblioteki,
		- w pliku `~/.profile` lub `~/.bashrc`:
			- `unset LD_PRELOAD` - co całkowicie usunie zmienną ze środowiska, 
			- `export LD_PRELOAD=` ustawienie zmiennej na domyślne biblioteki,
			- usunąć złośliwą bibliotekę.   
- `LD_LIBRARY_PATH`:
	- co określa ta zmienna: ścieżkę (listę katalogów), pod którą są wyszukiwane biblioteki współdzielone, z których korzystają programy.
	- zagrożenie: atakujący może podać fałszywą ścieżkę, w której umieści swoje biblioteki. To może spowodować, że standardowe biblioteki w ogóle się nie załadują. Biblioteki mogą zawierać dowolny kod, który będzie wykonywany po uruchomieniu polecenia. 
	- jak wykryć: sprawdzić miejsca gdzie ścieżka dla zmiennej mogła zostać ukryta, czyli:
		- `~/.bashrc`, 
		- `~/.profile`,
		- `/etc/security/pam_env.conf`,
		- `cat /proc/PID/environ | grep LD_LIBRARY_PATH` - sprawdzenie jakiej zmiennej używają procesy, 
		- czerwone flagi: ścieżka wskazuje na katalog użytkownika (np. `/home/user/lib`).
	- jak naprawić:  
		- w shell poleceniem `export LD_PRELOAD=...` lub w pliku `pam_env.conf`,
		- w pliku `~/.profile` lub `~/.bashrc`, należy zmienić zmienną za pomocą:
			- `unset LD_LIBRARY_PATH` - co całkowicie usunie zmienną ze środowiska, 
			- `export LD_LIBRARY_PATH= ` lub `LD_LIBRARY_PATH=/usr/local/lib:/usr/lib:/lib` - ustawienie zmiennej na standardowe biblioteki.  
			- usunąć złośliwą bibliotekę.  
			UWAGA: ścieżka ustawiona np. na katalog `/home/user/lib` - czerwona flaga. Biblioteki standardowe nie znajdują się w katalogu `/home`.
- `PATH`:
	- co określa ta zmienna: to lista katalogów, w których system szuka programów (np. ls, ps, cat, itp).
	- zagrożenie: atakujący może podmienić ścieżkę do katalogu programów, na swój katalog, którym ma własne programy, odpowiadające nazwom standardowym programom. Zamiast systemowego programu, wykonuje się złośliwy program atakującego.
	- jak wykryć: sprawdzić wartości zminnej w:
		- `/etc/security/pam_env.conf`,
		- `~/.bashrc`, `~/.profile`,
		- `~/.pam_environment`,
		- `/etc/profile`, `/etc/bash.bashrc`.   
		Pomocne polecenia:
		- `grep -r "PATH=" /etc/security/pam_env.conf /etc/profile /etc/bash.bashrc /home/*/.bashrc /home/*/.profile /root/.bashrc 2>/dev/null` - sprawdzenie w plikach konfiguracyjnych użytkowników i systemu,
		- `echo $PATH` - sprawdzenie aktualnej zmiennej w środowisku, 
		- `cat /proc/*/environ 2>/dev/null | grep "PATH=" | sort -u` - sprawdzenie w działających procesach czy nie ustawiono innego PATH przed uruchomieniem. 
		- czerwone flagi - podejrzane PATH:
			- zawiera `/tmp`, `/home,` `/var/tmp`, `/dev/shm`, 
			- zaczyna się od katalogu użytkownika (np. `/home/attacker/bin:`),
			- brak standardowych katalogów `/usr/bin`, `/bin`. 
	- jak naprawić: 
		- przywrócić bezpieczny PATH w pliku, w którym został zmieniony na podejrzany. Bezpieczny PATH:  
			- w pliku `/etc/security/pam_env.conf` - `PATH DEFAULT=/usr/local/bin:/usr/bin:/bin`,
			- w `~/.bashrc`, `~/.profile` - `export PATH=/usr/local/bin:/usr/bin:/bin`,
			- jeśli działa w systemie `~/.pam_environment` - usunąć plik i upewnić się, że `user_readenv=0` w PAM istnieje.
- `TMPDIR`:
	- co określa ta zmienna: wskazuje systemowi oraz programów, gdzie mają być tworzone pliki i katalogi tymczasowe, zamiast domyślnych `/tmp`, lub `/var/tmp`.  
	- zagrożenie: atakujący może ustawić TMPDIR na własny katalog, który może kontrolować. Dzięki czemu może:
		- przechwytywać pliki,
		- ponieważ programy zapisują a potem od razu kasują pliki w `/tmp`, gdy nie są już potrzebne, atakujący może go w ogóle nie zobaczyć. Jeśli jednak `TMPDIR` jest ustawiony na jego katalog, pliki zostają u niego na stałe. 
		- DoS - atakujący może ustawić `TMPDIR` na małą partycję a potem wypełnić ją,
		- atakujący może także po podmianie `TMPDIR` mając pełny dostęp do swojego katalogu, kasować tymczasowe pliki programów powodując ich nieprawidłowe wykonanie, 
		- atakujący może przechwycić nazwę pliku zapisywanego do `/tmp` przez program i po podmianie tego katalogu utworzyć plik o tej samej nazwie, powodując, że program go wykona. 
	- jak wykryć: sprawdzić ścieżki w:
		- `/etc/security/pam_env.conf`, 
		- `~/.bashrc`, `~/.profile`, 
		- `~/.pam_environment`, 
		- `/etc/profile`.  
		Przydatne polecenia:
			- `grep -r "TMPDIR" /etc/security/pam_env.conf /etc/profile /etc/bash.bashrc /home/*/.bashrc /home/*/.profile /root/.bashrc 2>/dev/null` - sprawdzenie w plikach konfiguracyjnych użytkowników i systemu,
			- `echo $TMPDIR` - sprawdzenie aktualnej zmiennej w środowisku, 
			- `cat /proc/*/environ 2>/dev/null | grep "TMPDIR" | sort -u`.
		- czerwone flagi:
			- zmienna wskazuje na katalog użytkownika, np. `/home/attacker/tmp`, `/tmp/atacker`,
			- zmienna wskazuje na katalog publiczny `/tmp`, `/var/tmp`. Może wskazywać na próbę wczesniejszej zmiany ścieżki.  
	- jak naprawić: poprawnie, zmienna nie powinna być ustawiona w ogóle, ale jeśli jest należy:
		- w pliku `/etc/security/pam_env.conf` ustawić `TMPDIR OVERRIDE=` (pusta wartość), lub usunąć wpis całkowicie, 
		- w plikach `~/.bashrc`, `~/.profile` usunąć linię z `TMPDIR=` lub dodać `unset TMPDIR`, 
		- w pliku `~/.pam_environment` usunąć plik i upewnić się, że `user_readenv=0` w PAM istnieje. 
- `IFS`:
	- co określa ta zmienna: (Internal Field Separator) określa jakie znaki są używane do dzielenia tekstu na osobne słowa. Domyślnie są to: spacja, tabulator, nowa linia.
	- zagrożenie:
		 - uszkodzenie skryptu - jeśli zmienne w skrypcie nie są ujęte w cudzysłów, to zmienna ta jest podatna na zmienną `IFS`,
		 - omijanie filtrów - atakujący może ustawić `IFS` na znak, który jest blokowany, (np. '/'), aby zapisać polecenie beze tego znaku np. `ls $IFStmp`,
		 - zmiana przeznaczenia skryptu - w połączeniu z podatnym skryptem, nie używającym zmiennych w cudzysłowach, atakujący może zmienić działanie skryptu, dzięki czemu będzie mógł wykradać dane lub eskalować uprawnienia.   
		 UWAGA: to jest rzadki wektor ataku, większość nowoczesnych systemów i dobrze napisanych skryptów jest na niego odporna (dzięki zastosowanym "").
	- jak wykryć: sprawdzić wartość zmiennej:
		-  `echo $IFS | cat -A` - jeśli nic nie zostanie wyświetlone, oznacza, że zmienna została poprawnie ustawiona na białe znaki, 
		- `grep -r "IFS" /etc/security/pam_env.conf /etc/profile /etc/bash.bashrc
	 /home*/.bashrc /home/*/.profile /root/.bashrc 2>/dev/null` - to polecenie sprawdzi pliki konfiguracyjne, w których zmienna mogła zostać podmieniona, 
		- `cat /proc/*/environ 2>/dev/null | grep "IFS"` - sprawdzenie procesów z jaką wartością zmiennej pracują.   
		czerwone flagi: 
			- jeśli jakikolwiek wpis `IFS=` w plikach konfiguracyjnych, 
			- `IFS` ustawiona na znak inny niż domyślne. 
	- jak naprawić:  
		- w pliku `pam_env.conf` - należy usunąć linie z `IFS=` lub ustawić na `IFS OVERRIDE= (pusta wartość)`,
		- w plikach `/.bashrc`, `/.profile` - należy usunąć linię z `IFS=` lub dodać wpis `unset IFS`,
		- w `~/.pam_environment` - najlepiej usunąć plik i upewnić się, że opcja `user_readenv=0`,
		- proces uruchomiony - należy go zabić (kill) - to zatrzyma proces, jednak nie usunie przyczyny. 
- `LOCALE`:
	- co określa ta zmienna: to cała grupa zmiennych odpowiadających za ustawienia lokalne, do tych zmiennych należą:
		- `LANG` - wszystkie kategorie (jeśli nie ustawiono nic bardziej szczegółowego), czyli komunikaty, daty, sortowanie, znaki,
		- `LC_CTYPE` - znaki - wielkość liter, czy znak jest cyfrą czy literą, 
		- `LC_COLLATE` - sortowanie (kolejność liter), czy ż jest po z,
		- `LC_TIME` - format daty i godziny, 
		- `LC_NUMERIC` - format liczb, (separator dziesiętny) czyli 3,14 vs 3.14,
		- `LC_MONETARY` - format waluty ($ vs zł),
		- `LC_MESSAGES` - język komunikatów, 
		- `LC_ALL` - nadpisuje wszystko, ma najwyższy priorytet. Dobrą praktyką jest ustawienie tej zmiennej na `LC_ALL=C`, wtedy system dostaje kompletne ustawienia lokalne i zachowuje się przewidywalnie i eliminuje skutki podmiany pozostałych zmiennych lokalnych. 
	- zagrożenie: największym zagrożeniem związanym ze zmianą powyższych zmiennych jest możliwość zburzenia działania skryptów i programów, które polegają na konkretnym formacie danych. Przykładowo zmiana języka może sprawić, że polecenia w skrypcie będą miały zupełnie inne znaczenie i skrypt nie wykona się właściwie. 
	- jak wykryć: należy szukać nietypowych wartości lub konfiguracji, które ułatwiają atak: 
		- `greep -r "LANG|LC_*" /etc/security/pam_env.conf /home/*/.bashrc /etc/profile 2>/dev/null` - za pomocą tego polecenia sprawdzamy występowanie zmiennych z nietypowymi wartościami (np. zawierającymi `../`, `:`), 
		- `grep AcceptEnv /etc/ssh/sshd_config` - czy `sshd_config` zawiera `AcceptEnv LANG LC_*`. W nowoczesnych systemach wyłączone domyślnie. 
		- `grep pam_env /etc/pam.d/sshd /etc/pam.d/common-session` - czy moduł `pam_env.so` czyta plik użytkownika (`user_readenv=`) - powinno być wyłączone. 
		- Czerwone flagi:
			- `AcceptEnv LANG LC_*` w pliku `sshd_config` - jeśli występuje w pliku `sshd_config` jako włączony, oznacza to, że jeśli klient wyśle swoje zmienne to zostaną zaakceptowane i ustawione w sesji na serwerze,
			- `user_readenv=1` dla modułu `pam_env.so`, to oznacza ze opcja jest włączona i moduł pozwoli użytkownikowi na ustawienie własnych, dodatkowych zmiennych środowiskowych za pomocą pliku `~/.pam_environment` w jego katalogu domowym, 
			- nietypowe wartości w `LANG/LC_*` w plikach konfiguracyjnych (zawierające znaki specjalne).
	- jak naprawić:
		- jeśli `sshd_config` akceptuje locale od klienta - należy wyłączyć `AcceptEnv LANG LC_*` lub zakomentować te linie. 
		- jeśli `pam_env.so` czyta plik użytkownika - należy ustawić `user_readenv=0` w wywołaniu `pam_env.so` w plikach PAM. 
		- podejrzane wartości w plikach konfiguracyjnych - należy usunąć lub zakomentować linie ustawiające `LANG/LC_*` w `pam_env.conf`, `.bashrc`, `.profile`. Użytkownicy nie powinni ustawiać tych zmiennych systemowo.
Wszystkie powyższe zmienne można monitorować automatycznie za pomocą programów, które mają reguły do wykrywania procesów korzystających ze zmiennymi:
	- EDR, 
	- SIEM,
	- auditd.

## Wnioski bezpieczeństwa
Konfiguracja modułu odpowiedzialnego za środowisko to poważny potencjalny wektor ataku, gdzie atakujący ma wiele możliwości manipulacji środowiskiem, dlatego też:
- należy monitorować zmienne środowiskowe, możliwa jest tu automatyzacja za pomocą programów EDR, SIEM, auditd. 
- należy monitorować konfigurację PAM typu common-session, jednak należy pamiętać, że pliki konfiguracyjne danej usługi mogą mieć w /etc/pam.d/ osobne lub dodatkowe ustawienia poza ustawieniami globalnymi, 
- należy weryfikować czy w konfiguracji PAM  (`sshd`, `common-session`) nie została ustawiona opcja `user_readenv=1` (np. `session required pam_env.so user_readenv`), najlepiej ją wyłączyć i opierać się tylko na pliku `pam_env.conf`,
- należy weryfikować czy w pliku `sshd_config` nie jest ustawiona opcja `AcceptEnv LANG LC_*` 