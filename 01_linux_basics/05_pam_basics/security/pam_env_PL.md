# /etc/security/pam_env.conf

## Cel
Zapoznanie się z opcjami konfiguracyjnymi modułu `pam_env.so` znajdującymi się w pliku konfiguracyjnym `/etc/security/pam_env.conf`.

## Do czego służy moduł pam_env.so
Jest to moduł ustawiający zmienne środowiskowe na podstawie:
- `/etc/security/pam_env.conf`,
- `/etc/environment`.
- opcjonalnie w pliku użytkownika `~/.pam_environment`.

## Jaka jest różnica pomiędzy plikami `/etc/security/pam_env.conf` i `/etc/environment`
- `/etc/security/pam_env.conf` - to zaawansowany, warunkowy i elastyczny plik konfiguracji modułu `pam_env.so`. Oferuje on znacznie większe możliwości niż `/etc/environment` ale jednocześnie ma bardziej skomplikowaną składnię. Jest on bardzo przydatny gdy potrzeba logiki, np.:
	- ustaw zmienną `PATH` na `X`, lecz jeśli zmienna `Y` jest już ustawiona to zrób coś innego,
	- ustaw zmienną `JAVA_HOME` na wartość wynikającą z innej zmiennej (np. $(HOME)/jdk).
	Uwaga: zmienne `user` i `HOME` mogą nie być dostępne w momencie działania `pam_env`, więc może to nie zadziałać w praktyce.
- `/etc/environment` - to plik służący do ustawiania prostych zmiennych środowiskowych w formacie KEY=VAL dla wszystkich użytkowników i wszystkich procesów. Ma prostą składnię w której każda linia to para NAZWA=wartość. Nadaje się dobrze do ustawiania zmiennych, które powinny być dostępne dla całego systemu, niezależnie od sposobu logowania użytkownika (konsola, SSH, GUI). UWAGA: jeśli w konfiguracji PAM (`sshd`, `common-session`) jest włączone `user_readenv=1`, to zwykły użytkownik może ustawić własne zmienne środowiskowe, a to stwarza cały szereg zagrożeń w tym wstrzyknięcie biblioteki, podmianę PATH itp. Ustawienie opcji `user_readenv=0` to kompletne i całkowite wyłączenie możliwości użycia pliku `~/.pam_environment`.

## Dlaczego dwa pliki dla jednego modułu
Ponieważ moduł `pam_env.so` działa w kilku warstwach:
- Warstwa 1 - w pierwszej kolejności są czytane reguły z pliku `/etc/security/pam_env.conf`, ponieważ to tutaj definiuje się skomplikowaną logikę,
- Warstwa 2 - następnie, moduł odczytuje plik `/etc/environment` (można to wyłączyć opcją `readenv=0`). Są tutaj proste pary, a każda z nich ustawia zmienną. 
- Warstwa 3 - istnieje też możliwość by moduł czytał również pliki z `~/.pam_environment`, jednak jest to potencjalnie duże ryzyko bezpieczeństwa, dlatego opcja `user_readenv` jest domyślnie wyłączona w nowoczesnych systemach. 

## Składnia `pam_env.conf`
Należy pamiętać, że w składni kluczowe są składniki takie jak DEFAULT i OVERRIDE:
- `DEFAULT` - ustawi zmienną jeśli nie została jeszcze ustawiona, np. `VAR DEFAULT=wartość`. Moduł `pam_env.so` sprawdzi czy zmienna o podanej nazwie już istnieje w środowisku (np. została ustawiona w `/.profile`, `/.bashrc` lub przez system). Jeśli nie istnieje, moduł ustawi ją na podaną wartość, jeśli istnieje zostawi ją bez zmian. 
- `OVERRIDE` - zawsze nadpisuje zmienną, nawet jeśli już została ustawiona. `OVERRIDE` ma najwyższy priorytet.   
Przykład ustawienie PATH:  
`PATH OVERRIDE=/usr/local/bin:/usr/bin:/bin:/opt/bin`  

## Ważne parametry `pam_env.conf` i zagrożenia z nimi związane
- `LD_PRELOAD`: 
	- co określa ta zmienna: to ścieżka wskazująca, która biblioteka ma być załadowana jako pierwsza (przed standardowymi).
	- zagrożenie: wstrzyknięcie biblioteki przez atakującego. Jak to działa:
		- atakujący ustawia ścieżkę do biblioteki, którą umieścił podczas włamania na konto np: `export LD_PRELOAD=/home/biblioteka/moja_biblioteka.so`. 
		- system (loader ld.so) widzi tę zmienną i zanim zrobi cokolwiek innego, ładuje tą bibliotekę do pamięci,
		- w tej bibliotece jest funkcja o tej samej nazwie co oryginalna funkcja systemowa (np. `open()`, `write()`),
		- wtedy gdy program próbuje zapisać coś, zamiast oryginalnej funkcji wykonuje się kod z biblioteki, którą wstrzyknął. Kod w bibliotece praktycznie może być dowolny, to stwarza bardzo duże zagrożenie.   
		UWAGA: atakujący może ukryć tę zmienną w 3 różnych miejscach: `~/.bashrc`, `~/.profile` i `/etc/security/pam_env.conf`.
	- jak wykryć: 
		- sprawdzić miejsca gdzie ścieżka dla zmiennej mogła zostać ukryta, czyli:
			- `~/.bashrc`, 
			- `~/.profile`,
			- `/etc/security/pam_env.conf`.
	- jak naprawić: należy zmienić ścieżkę do biblioteki w miejscu, w którym została zmieniona. Jeśli zmiana zmiennej nastąpiła w:
		- shell poleceniem `export LD_PRELOAD=...` lub w pliku `pam_env.conf` to zmiana w pliku `/etc/security/pam_env.conf`, nadpisze tę zmienną ustawiając poprawną ścieżkę, za pomocą zapisu `LD_PRELOAD OVERRIDE=`. Pusta wartość ustawi standardowe biblioteki,
		- pliku `~/.profile` lub `~/.bashrc`, należy zmienić zmienną za pomocą:
			- `unset LD_PRELOAD` - co całkowicie usunie zmienną ze środowiska, 
			- `export LD_PRELOAD=` ustawienie zmiennej w ten sposób sprawi, że będą się ładować tylko standardowe biblioteki.  
			- usunąć złośliwą bibliotekę.  
		UWAGA: Jeśli zmienna `LD_PRELOAD` została ustawiona przez atakującego w plikach `~/.profile` lub `~/.bashrc`, to zmiana w pliku `/etc/security/pam_env.conf` nie pomoże. Pliki te ładują się po PAM, więc zmienna zostanie ponownie nadpisana przez nie w systemie. 
- `LD_LIBRARY_PATH`:
	- co określa ta zmienna: ścieżkę (listę katalogów), pod którą są wyszukiwane biblioteki współdzielone, z których korzystają programy.
	- zagrożenie: atakujący może podać fałszywą ścieżkę, w której umieści swoje biblioteki. To może spowodować, że standardowe biblioteki w ogóle się nie załadują. Jest to nieco łatwiejsze do wykrycia. Podobnie jak w zagrożeniu `LD_PRELOAD`, tutaj również biblioteki mogą zawierać dowolny kod, który będzie wykonywany po uruchomieniu polecenia. Np. jeśli złośliwa biblioteka ma funkcję ls, której kod ma za zadanie np. usuniecie danych z dysku, to użytkownik myśląc, że wykonuje niewinne polecenie, które ma mu pokazać zawartość katalogu, usunie dane. Należy tu zwrócić uwagę, że biblioteka to nie program, jednak każdy program korzysta z bibliotek i zapisanych w nich funkcji. W przypadku ls, program korzysta z funkcję biblioteczną readdir() i to właśnie funkcja jest podmieniana a nie program. 
	- jak wykryć: sprawdzić miejsca gdzie ścieżka dla zmiennej mogła zostać ukryta, czyli:
		- `~/.bashrc`, 
		- `~/.profile`,
		- `/etc/security/pam_env.conf`,
		- `cat /proc/PID/environ | grep LD_LIBRARY_PATH` - sprawdzenie jakiej zmiennej używają procesy, 
		- czerwone flagi: ścieżka wskazuje na katalog użytkownika (np. `/home/user/lib`).
	- jak naprawić: należy zmienić ścieżkę do biblioteki w miejscu, w którym została zmieniona. Jeśli zmiana zmiennej nastąpiła w:  
		- shell poleceniem `export LD_PRELOAD=...` lub w pliku `pam_env.conf` to zmiana w pliku `/etc/security/pam_env.conf`, nadpisze tą zmienną ustawiając poprawną ścieżkę, za pomocą zapisu `LD_LIBRARY_PATH OVERRIDE=`.
		- pliku `~/.profile` lub `~/.bashrc`, należy zmienić zmienną za pomocą:
			- `unset LD_LIBRARY_PATH` - co całkowicie usunie zmienną ze środowiska, 
			- `export LD_LIBRARY_PATH= ` lub `LD_LIBRARY_PATH=/usr/local/lib:/usr/lib:/lib` - ustawienie zmiennej na standardowe biblioteki.  
			- usunąć złośliwą bibliotekę.  
			UWAGA: ścieżka ustawiona np. na katalog `/home/user/lib` - czerwona flaga. Biblioteki standardowe nie znajdują się w katalogu `/home`.
- `PATH`:
	- co określa ta zmienna: to lista katalogów, w których system szuka programów (np. ls, ps, cat, itp).
	- zagrożenie: atakujący może podmienić ścieżkę do katalogu programów, na swój katalog, którym ma własne programy, odpowiadające nazwom standardowym programom. Programy napisane przez atakującego mogą korzystać z zupełnie innych funkcji bibliotek standardowych, opierać się wyłącznie na własnym kodzie lub też korzystać z bibliotek podmienionych przez atakującego, choć ta ostatnia opcja wymaga dodatkowej podmiany bibliotek. Jak taki atak działa:
		- atakujący tworzy fałszywe programy (np. `ls`, `ps`, `sudo`, `ssh`),
		- umieszcza je w katalogu, do którego ma dostęp (np. `/home/haker/bin/`),
		- dodaje swój katalog na początek `PATH` (np. `PATH=/home/haker/bin:$PATH`),
		- gdy użytkownik lub skrypt użyje takiego programu, system wykonuje fałszywy program, bo ścieżka do niego jest pierwsza w `PATH`,
		- efekt - zamiast systemowego programu, wykonuje się program atakującego co pozwala na wykonanie dowolnego kodu.  
	- jak wykryć: należy monitorować:
		- `/etc/security/pam_env.conf` - ma globalny zakres działania, jednak zmiana wymaga roota, 
		- `~/.bashrc`, `~/.profile` - dotyczy danego użytkownika, atakujący po włamaniu na konto może łatwo zmienić ścieżkę w tych plikach,
		- `~/.pam_environment` - dotyczy danego użytkownika, rzadko się zdarza ponieważ nowoczesne systemy mają wyłączony ten plik,
		- `/etc/profile`, `/etc/bash.bashrc` - zakres globalny, jednak wymaga dostępu do root.   
		Pomocne polecenia:
		- `grep -r "PATH=" /etc/security/pam_env.conf /etc/profile /etc/bash.bashrc /home/*/.bashrc /home/*/.profile /root/.bashrc 2>/dev/null` - sprawdzenie w plikach konfiguracyjnych użytkowników i systemu,
		- `echo $PATH` - sprawdzenie aktualnej zmiennej w środowisku, 
		- `cat /proc/*/environ 2>/dev/null | grep "PATH=" | sort -u` - sprawdzenie w działających procesach czy nie ustawiono innego PATH przed uruchomieniem. 
		** Wyjaśnienie zapisu `2>/dev/null`:  
		Numer `2` to numer deskryptora dla stderr - standardowe wyjście błędów. 
			- 1 - stdout - normalne wyjście, 
			- 2 - stderr - błędy.
		Polecenie przed tym zapisem generuje dużo błędów na "permission denied" dla katalogów innych użytkowników, "no such file" jeśli proces zakończył się. Przekierowując tym zapisem, można ukryć te błędy i widzieć tylko poprawne wyniki, czyli zawartość environ dla procesów, do których jest dostęp. Plik `/dev/null` natychmiast usuwa wszystko, co do niego trafia. 
		- czerwone flagi - podejrzane PATH:
			- zawiera `/tmp`, `/home,` `/var/tmp`, `/dev/shm`, 
			- zaczyna się od katalogu użytkownika (np. `/home/attacker/bin:`),
			- brak standardowych katalogów `/usr/bin`, `/bin`. 
	- jak naprawić: 
		- przywrócić bezpieczny PATH w pliku, w którym został zmieniony na podejrzany. Bezpieczny PATH:  
			- dla pliku `/etc/security/pam_env.conf` - `PATH DEFAULT=/usr/local/bin:/usr/bin:/bin`,
			- dla `~/.bashrc`, `~/.profile` - `export PATH=/usr/local/bin:/usr/bin:/bin`,
			- jeśli działa w systemie `~/.pam_environment` - usunąć plik i upewnić się, że `user_readenv=0` w PAM istnieje.
- `TMPDIR`:
	- co określa ta zmienna: wskazuje systemowi oraz programów, gdzie mają być tworzone pliki i katalogi tymczasowe, zamiast domyślnych `/tmp`, lub `/var/tmp`. Oznacza to, że jeśli TMPDIR nie jest ustawiony, to zostaną użyte domyślne. Pliki tymczasowe są tworzone przez programy, z określonymi uprawnieniami w folderze domyślnym lub wskazanym przez zmienną a następnie gdy nie są już potrzebne są kasowane. Uprawnienia domyślnych katalogów są ustawione na 1777 - tzn. że każdy może czytać, zapisywać i wykonywać w tym katalogu pliki, ale dzięki sticky bit, tylko właściciel może usuwać swoje pliki. 
	- zagrożenie: atakujący może ustawić TMPDIR na własny katalog, który może kontrolować. Dzięki czemu może:
		- przechwytywać pliki - czasami programy tworzą plik w `/tmp` z uprawnieniami 600 a to oznacza, że tylko właściciel pliku może go przeczytać i zapisać. Takie pliki czasami zawierają wrażliwe dane np. hasła w jawnej postaci. Jeśli `TMPDIR` zostanie podmieniony na katalog atakującego, to on ma pełną kontrolę nad katalogiem - może przechwytywać, podmieniać lub usuwać tworzone tam pliki tymczasowe. 
		- ponieważ programy zapisują a potem od razu kasują pliki w `/tmp`, gdy nie są już potrzebne, atakujący może go w ogóle nie zobaczyć. Jeśli jednak `TMPDIR` jest ustawiony na jego katalog, pliki zostają u niego na stałe. 
		- DoS - atakujący może ustawić `TMPDIR` na małą partycję a potem wypełnić ją, co powoduje, że program nie może zapisać swoich plików tymczasowych i nie wykona się prawidłowo, 
		- atakujący może także po podmianie `TMPDIR` mając pełny dostęp do swojego katalogu, kasować tymczasowe pliki programów powodując ich nieprawidłowe wykonanie, 
		- atakujący może przechwycić nazwę pliku zapisywanego do `/tmp` przez program i po podmianie tego katalogu utworzyć plik o tej samej nazwie, powodując, że program go wykona. 
	- jak wykryć: należy monitorować miejsca, w których atakujący mógł podmienić `TMPDIR`:
		- `/etc/security/pam_env.conf` - zakres globalny, wymaga roota, 
		- `~/.bashrc`, `~/.profile` - zakres dla danego użytkownika, wystarczy, że atakujący włamie się na jego konto, 
		- `~/.pam_environment` - zakres dla danego użytkownika, zwykle wyłączone w nowoczesnych systemach, 
		- `/etc/profile` - zakres globalny, ale wymaga dostępu do root.  
		Przydatne polecenia:
			- `grep -r "TMPDIR" /etc/security/pam_env.conf /etc/profile /etc/bash.bashrc /home/*/.bashrc /home/*/.profile /root/.bashrc 2>/dev/null` - sprawdzenie w plikach konfiguracyjnych użytkowników i systemu,
			- `echo $TMPDIR` - sprawdzenie aktualnej zmiennej w środowisku, 
			- `cat /proc/*/environ 2>/dev/null | grep "TMPDIR" | sort -u`.
		- czerwone flagi:
			- zmienna wskazuje na katalog użytkownika, np. `/home/attacker/tmp`, `/tmp/atacker`,
			- zmienna wskazuje na katalog publiczny `/tmp`, `/var/tmp`. Choć nie jest to podejrzane samo w sobie, to jednak jawne ustawienie zmiennej na te katalogi podczas gdy są one domyślne bez ustawiania zmiennej jest podejrzane, ponieważ może to oznaczać zacieranie śladów, po niewłaściwej ścieżce użytej podczas ataku. 
	- jak naprawić: poprawnie, zmienna nie powinna być ustawiona w ogóle, ale jeśli jest należy:
		- w pliku `/etc/security/pam_env.conf` ustawić `TMPDIR OVERRIDE=` (pusta wartość), lub usunąć wpis całkowicie, 
		- w plikach `~/.bashrc`, `~/.profile` usunąć linię z `TMPDIR=` lub dodać `unset TMPDIR`, 
		- w pliku `~/.pam_environment` usunąć plik i upewnić się, że `user_readenv=0` w PAM istnieje. 
- `IFS`:
	- co określa ta zmienna: (Internal Field Separator) określa jakie znaki są używane do dzielenia tekstu na osobne słowa. Domyślnie są to: spacja, tabulator, nowa linia.
	- zagrożenie:
		 - uszkodzenie skryptu - jeśli zmienne w skrypcie nie są ujęte w cudzysłów, to zmienna ta jest podatna na zmienną `IFS`. To oznacza, że podstawiając tą zmienna w jakimś miejscu skryptu zostanie podzielona ona wg separatorów określonych z zmiennej `IFS`. Zmiana zmiennej `IFS` może zmienić sposób interpretacji danych (podzielić słowa inaczej) co może spowodować niepoprawne działanie skryptu. `Dlatego bezpieczną praktyką, jest używanie w skryptach zmiennych ujętych w cudzysłów`, nie są one wtedy podatne na działanie zmiennej `IFS` a słowa nie są dzielone przez zmieniony separator.
		 - omijanie filtrów - atakujący może ustawić `IFS` na znak, który jest blokowany, (np. '/'), aby zapisać polecenie beze tego znaku np. `ls $IFStmp`,
		 - zmiana przeznaczenia skryptu - w połączeniu z podatnym skryptem, nie używającym zmiennych w cudzysłowach, atakujący może zmienić działanie skryptu, dzięki czemu będzie mógł wykradać dane lub eskalować uprawnienia.   
		 UWAGA: to jest rzadki wektor ataku, większość nowoczesnych systemów i dobrze napisanych skryptów jest na niego odporna (dzięki zastosowanym "").
	- jak wykryć: 
		- najprostszym sposobem sprawdzenia zmiennej jest polecenie `echo $IFS | cat -A` - jeśli nic nie zostanie wyświetlone, oznacza, że zmienna została poprawnie ustawiona na białe znaki, 
		- `grep -r "IFS" /etc/security/pam_env.conf /etc/profile /etc/bash.bashrc /home*/.bashrc /home/*/.profile /root/.bashrc 2>/dev/null` - to polecenie sprawdzi pliki konfiguracyjne, w których zmienna mogła zostać podmieniona, 
		- `cat /proc/*/environ 2>/dev/null | grep "IFS"` - sprawdzenie procesów z jaką wartością zmiennej pracują.   
		CZERWONA FLAGA: 
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
		- `grep -r "LANG|LC_*" /etc/security/pam_env.conf /home/*/.bashrc /etc/profile 2>/dev/null` - za pomocą tego polecenia sprawdzamy występowanie zmiennych z nietypowymi wartościami (np. zawierającymi `../`, `:`), 
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