# tmp_analysis

## Cel 
Zrozumienie zagrożeń związanych z katalogami tymczasowymi oraz metod monitorowania i bezpiecznego czyszczenia. 

## Czym są katalogi tymczasowe i po co istnieją 
Programy i system często potrzebują tworzyć pliki tymczasowe, np:
- sesje użytkowników, 
- cache - tymczasowe dane, które mogą być odtworzone,
- locki - informacje, że jakiś program działa, żeby nie uruchomić go drugi raz, 
- sockety, FIFO, named pipes - pliki służące do komunikacji między procesami. 
Te pliki nie muszą być przechowywane na stałe, po zrestartowaniu systemy nie są już potrzebne. Właśnie do tego służą katalogi tymczasowe:
- `/tmp`,
- `/var/tmp`,
- `/dev/shm`.

## Czym się różnią katalogi `/tmp`, `/var/tmp` i `/dev/shm`

### /tmp
To katalog przechowujący pliki tymczasowe dla aplikacji i użytkowników. W zależności od konfiguracji może być zlokalizowany na dysku lub w pamięci RAM. Zawartość katalogu `/tmp` może, ale nie musi być usuwana po restarcie systemu - to zależy od konfiguracji systemu (np. tmpfs, systemd-tmpfiles). 
- Mechanizmy czyszczenia katalogu `/tmp` w Linux
W nowoczesnych systemach Linux występują dwa mechanizmy czyszczenia katalogu `/tmp`:
	- `tmpfs` - tmpfs to wirtualny system plików, który przechowuje dane w pamięci RAM, a nie na dysku. Oznacza to, że pliki `tmpfs` znikają po restarcie systemu (bo RAM się czyści). `/tmp` może być włączony jako tmpfs w `/etc/fstab` np.:   
		`tmpfs /tmp tmpfs defaults,notime,mode=1777 0  0`   
	Daje to szybki dostęp i automatyczne czyszczenie po restarcie. Sprawdzenie, czy `/tmp` jest w tmpfs:
		- `mount | grep /tmp` - pokaże wszystkie zamontowane systemy plików i ich opcje. Jeśli `/tmp` jest `tmpfs`, to można zobaczyć zapis:
		`tmpfs on /tmp type tmpfs (rw,nosuid,nodev)`
		Jeśli nic nie ma, `/tmp` jest zwykłym katalogiem na dysku np: ext4.
		- `df -hT /tmp`
	- `systemd-tmpfiles` to mechanizm zarządzany przez systemd, który automatycznie czyści stare pliki w katalogach tymczasowych. Konfiguracja znajduje się w plikach:
		- `/etc/tmpfiles.d/*.conf` - stawienia globalne, miejsce, w którym można samodzielnie stworzyć reguły, które nadpiszą domyślne ustawienia,
		- `/usr/lib/tmpfiles.d/*.conf` - ustawienia domyślne, dostarczane przez dystrybutora (nie należy go modyfikować).  
		Przykład:   
		Tworzymy plik o nazwie `/etc/tmpfiles.d/tmp.conf` a w nim ustawiamy regułę by pliki `/tmp` były czyszczone co 7 dni:
			`q /tmp 1777 root root 7d`
		Sprawdzanie czy tmpfiles działa w systemie:    
			`systemctl status systemd-tmpfiles-clean.timer`   
		To pozwala zobaczyć, kiedy ostatnio działało i czy jest aktywne. Aby wymusić czyszczenie należy użyć polecenia:
			`sudo systemd-tmpfiles --clean`  
`/tmp` jest katalogiem otwartym dla wszystkich użytkowników, dlatego wymaga odpowiednich uprawnień i mechanizmów ochronnych 1777 (sticky bit). 

### /var/tmp
Przechowuje pliki tymczasowe, które muszą przetrwać restart (np. dane tymczasowe aplikacji, które są odtwarzane po restarcie, ale lepiej ich nie tracić). Katalog `/var/tmp` znajduje się na dysku (nie w pamięci RAM). Czyszczenie katalogu odbywa się wyłącznie za pomocą `systemd-tmpfiles`. Można do pliku `/etc/tmpfiles.d/tmp.conf` dodać regułę dla czyszczenia katalogu `/var/tmp`, analogicznie jak dla katalogu `/tmp`. Uprawnienia katalogu to `drwxrwxrwt` lub `rwxrwxr-x` (często bez sticky bit, on nie jest tu potrzebny ponieważ `/var/tmp` nie jest tak intensywnie współdzielony między użytkownikami jak `/tmp`).

### /dev/shm
Jest to pamięć współdzielona (shared memory) - mechanizm do szybkiej wymiany danych między procesami. Pliki znajdują się wyłącznie w pamięci RAM, nie na dysku. Czyści się automatycznie podczas restartu, jednak można dodać dla niego regułę, podobnie jak dla `/tmp` i `/var/tmp` w pliku `/etc/tmpfiles.d/tmp.conf`. Uprawnienia tego katalogu to `rwxrwxrwt`, choć sticky bit nie jest wymagany, ponieważ cała zawartość katalogu znika po restarcie. To częsty wektor ataków, ponieważ admini często o nim zapominają, każdy ma prawo tworzyć i usuwać w nim pliki, a jeśli system nie ma `tmpfs` na `/tmp`, to `/dev/shm` jest jedynym katalogiem w RAM, gdzie można szybko utworzyć plik bez zapisu na dysk.

## SELinux i AppArmor
W niektórych systemach Linux dodatkową warstwę bezpieczeństwa zapewniają mechanizmy 'Mandatory Access Control' (MAC), takie jak SELinux lub AppArmor. To mechanizmy kontroli dostępu (nie tylko uprawnienia `rwx`). Mogą ograniczyć, które procesy mogą tworzyć/wykonywać pliki w `/tmp`.
- Rodzaje mechanizmów: 
	- SELinux (Security-Enhanced Linux) - jest systemem kontroli dostępu rozwijanym przez NSA i używanym głównie w systemach rodziny Red Hat (RHEL, Fedora, CentOS). W SELinux każdy plik i proces posiada specjalny kontekst bezpieczeństwa, który określa, jakie operacje są dozwolone. Konteksty te można wyświetlić poleceniem:  
		`ls -Z`   
	jeśli SELinux jest aktywny, obok standardowych uprawnień pojawia się dodatkowa kolumna z kontekstem bezpieczeństwa. 
	- AppArmor - w systemach takich jak Debian, Ubuntu zamiast SELinux często używany jest mechanizm AppArmor. AppArmor również ogranicza możliwości procesów, ale robi to poprzez profile przypisane do konkretnych aplikacji. Profile te określają, do jakich plików lub katalogów dana aplikacja może mieć dostęp. 
- W trakcie analiz incydentów, warto sprawdzić czy usługa SELinux lub AppArmor są uruchomione w systemie, ponieważ malware czasami próbuje je wyłączyć. 
	- Polecenia sprawdzające status AppArmor:
		- `aa-status` - to polecenie pokazuje czy moduł jest załadowany, czy są profile oraz w jakim trybie są (enforce lub complain),
		- `systemctl status apparmor` - polecenie sprawdza czy usługa jest włączona, czy jest uruchomiona oraz czy startuje automatycznie przy boot.
	- Polecenia sprawdzające status SELinux:
		- `getenforce` - szybkie sprawdzenie dające wynik Enforcing, Permissive lub Disabled,
		- `sestatus` - pełny status, pokazuje więcej informacji, np.:
			- SELinux status: enabled,
			- Current mode: enforcing,
			- Policy: targeted.

## Bezpieczne czyszczenie katalogu /tmp
- restart systemu - to działa w przypadku analizowanego systemu Kali,
- sudo rm -rf /tmp/* - to ręczne czyszczenie, które trzeba przeprowadzić bardzo ostrożnie i najlepiej w czasie, gdy system nie wykonuje krytycznych operacji. Ważna uwaga: niektóre aplikacje mogą trzymać otwarte deskryptory plików w /tmp - ręczne czyszczenie może powodować błędy aplikacji. UWAGA: polecenie to nie usuwa ukrytych plików,
- usuwanie plików starszych niż X dni, przy pomocy systemd-tmpfiles,

## Czego nie robić
- NIE usuwać katalogu /tmp,
- NIE czyścić katalogu podczas instalacji lub aktualizacji. 

## Wnioski bezpieczeństwa
- `tmpfs` - ma kilka zalet:
	- malware trudniej zostawić trwałe pliki,
	- stare pliki tymczasowe nie zalegają w systemie,
	- zmniejsza się powierzchnia ataku.
- `systemd-tmpfiles` jest szczególnie pomocne, gdy system działa wiele dni bez restartu. Bez ponownego uruchomienia przy ustawieniu wyłącznie tmpfs katalog długo by nie był czyszczony. `Tmpfiles` usuwa pliki starsze niż X dni, nawet gdy nie następuje restart systemu.
- ustawienie `sticky bit` dla katalogu `/tmp` zapewnia, że nikt nie może ingerować w pliki innego użytkownika, a to zmniejsza wektor ataków. Eliminuje to np. modyfikację pliku innego użytkownika umieszczając w nim złośliwe oprogramowanie lub usuwanie pliku, który jest w danej chwili używany przez aplikację.
- polecenie `sudo systemd-tmpfiles --clean` nie usuwa wszystkich plików ponieważ:
	- `systemd-tmpfiles` działa na podstawie wieku plików (mtime), a nie ich liczby czy rozmiaru, 
	- jeśli plik ma datę modyfikacji z dzisiaj, nie zostanie usunięty,
	- to jest celowe - system nie kasuje 'aktywnego' pliku w trakcie używania. 
- Ze względu na otwarty charakter katalogów `/tmp`, `/var/tmp`, `/dev/shm`, w których każdy użytkownik systemu może tworzyć pliki, warto regularnie monitorować ich zawartość pod kątem potencjalnych oznak nadużyć lub złośliwej aktywności. Jednym z prostych sposobów wstępnej analizy jest wyszukiwanie plików, które pozostają w katalogu dłużej niż powinny. Przykładowe polecenia:
	- `find /tmp -type f -mtime +7 -ls`
	- `lsof +D /tmp` - polecenie lsof (List Open Files) wyświetla listę plików aktualnie otwartych przez procesy w systemie, opcja +D `/tmp` ogranicza wynik do katalogu `/tmp`. Może to pomóc zidentyfikować aplikacje lub procesy, które aktywnie korzystają z plików tymczasowych. W kontekście analizy bezpieczeństwa pozwala to sprawdzić, czy nieznany proces nie wykorzystuje katalogu /tmp do uruchamiania lub przechowywania plików tymczasowych. 
- Regularna analiza zawartości katalogów `/tmp`, `/var/tmp`, `/dev/shm` jest jedną z podstawowych technik wstępnej diagnostyki systemu oraz elementem szybkiego triage w pracy analityka SOC. 
- SOC powinien zawsze uwzględnić czas UTC przy analizie plików tymczasowych, żeby timeline był spójny. 
- Mechanizmy takie jak SELinux lub AppArmor mogą ograniczać sposób, w jaki aplikacje korzystają z katalogów tymczasowych takich jak `/tmp`. Dzięki temu nawet jeśli złośliwe oprogramowanie zostanie uruchomione, to jego możliwości działania mogą być częściowo ograniczone przez politykę bezpieczeństwa systemu. W analizowanym systemie Kali Linux SELinux nie był zainstalowany, a AppArmor był wyłączony, dlatego żaden z nich nie wpływał na zachowanie katalogu `/tmp` w trakcie przeprowadzonego laboratorium.
- Katalogi `/tmp`, `/var/tmp`, `/dev/shm` są często wykorzystywane przez złośliwe oprogramowanie jako miejsce przechowywania tymczasowych plików. Wynika to z faktu, że katalogi są zapisywalne dla wszystkich użytkowników systemu. Atakujący mogą zapisywać w katalogach tych pliki wykonywalne lub tzw. loadery, które następnie pobierają właściwe złośliwe oprogramowanie i zapisują je w innych lokalizacjach systemu.


