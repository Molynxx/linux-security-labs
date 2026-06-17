# Boot_process_basics

## Cel 
Zrozumienie jak działa proces uruchamiania systemu i jakie znaczenie ma ta znajomość dla wykrywania backdoorów, które ładują się przed systemem. 

## Czym jest boot process
Jest to sekwencja zdarzeń od momentu włączenia zasilania komputera do momentu gdy pokazuje się panel logowania. Znajomość tego procesu jest ważna dla SOC, ponieważ narzędzia takie jak `ps` i `ls` nie widzą backdoorów uruchamiających się przed systemem. 

## Etapy bootowania
- `BIOS/UEFI` - na tym etapie sprawdzane jest poprawne działanie sprzętu (POST - Power-On Self Test), jeśli wszystko jest w porządku, płyta główna nie wydaje ostrzeżenia o błędzie -> szuka bootloadera na dysku,
- `Bootloader (GRUB)` - ładuje jądro i initramfs, pokazuje menu do wyboru systemu,
- `Kernel` - Jądro przejmuje kontrolę, inicjalizuje sprzęt oraz montuje initramfs,
- `initramfs` - tymczasowy system plików w pamięci, są ładowane sterowniki i montowane właściwe systemy plików, 
- `init / systemd` - rozpoczyna się pierwszy proces (PID 1), który następnie uruchamia usługi (sieć, SSH, logi). 

## Najważniejsze zagrożenia dla SOC i sposoby ich wykrywania
- `GRUB`:
	- zagrożenie: Atakujący posiada dostęp do konta root lub dostęp bezpośredni do konsoli i  edytuje `/etc/default/grub`, dodając `init=/bin/bash` do linii `GRUB_CMDLINE_LINUX_DEFAULT`. Następnie uruchamia `update-grub` (Debian/Ubuntu) i uruchamia ponownie system. Dzięki tym zabiegom po restarcie system nie uruchamia init/systemd lecz od razu powłokę bash jako  root bez logowania i bez haseł. Praktycznie ten atak jest skuteczny tylko z fizycznym dostępem lub przez zdalne zarządzanie (iLO/iDRAC), ponieważ zdalne połączenie nie będzie możliwe bo system nie uruchomi sieci.
	- jak wykryć: należy sprawdzić `/etc/default/grub` oraz `/boot/grub/grub.cfg` czy nie zawiera wpisu `init=/bin/bash`. Można wykorzystać polecenie: ` cat /etc/default/grub | grep "init="`. 
	- jak naprawić: 
		- usunąć `init=/bin/bash` z `/etc/default/grub`,
		- uruchomić `sudo update-grub` (Debian / Ubuntu),
		- zrestartować system, zmienić hasła i sprawdzić inne backdoory, często atakujący zostawia sobie dodatkowe.
- `initramfs`:
	- zagrożenie: jest to tymczasowy system plików, który jest ładowany do pamięci RAM razem z jądrem. Zawiera skrypty oraz sterowniki niezbędne do zamontowania właściwego systemu plików z dysku. `initramfs` nie jest plikiem, to jest archiwum, zawierające cały katalog ze skryptami i programami. Jego główny skrypt startowy nosi nazwę `init` i wykonuje się jako pierwszy. Atakujący (mający dostęp do root) może:  
		- rozpakować archiwum initramfs,
		- czytać rozpakowaną zawartość pliku (na niektórych systemach jest `initramfs.img`, a nie `initrd.img`),
		- dodać swój skrypt, 
		- spakować i zastąpić plik.
	Powyższy kod wykonuje się przed uruchomieniem systemu i jest niewidoczny dla `ps`, `ls`, `netstat` bo system nie jest jeszcze uruchomiony. Wykonuje połączenie z adresem ustalonym przez atakującego, dlatego nie widać również żadnych połączeń przychodzących, które mogłyby wzbudzić podejrzenia. Nie wymaga fizycznego dostępu. 
	- jak wykryć: najlepiej użyć do tego polecenia lsinitramfs, które wyświetla listę plików znajdujących się w initramfs bez rozpakowywania. Przykładowe polecenia: 
		- `lsinitramfs /boot/initrd.img-$(uname -r) | head -30` - sprawdzenie czy initramfs zawiera nieznane pliki,
		- `lsinitramfs /boot/initrd.img-$(uname -r) | grep -E "\.sh|\.so"` - szukanie skryptów i bibliotek.
	- jak naprawić: 
		- najprostszym rozwiązaniem jest przeinstalowanie pakietu jądra za pomocą polecenia `sudo apt install --reinstall linux-image-$(uname -r)` (Ubuntu /  Debian),
		- można także naprawić ręcznie - rozpakować archiwum, usunąć podejrzane pliki a następnie spakować. 
- `kernel`:
	- zagrożenie: podmiana jądra systemu (rootkit jądra). Atakujący jeśli ma dostęp do roota:
		- kompiluje własne jądro lub pobiera gotowe,
		- zastępuje `/boot/vmlinuz-$(uname -r)` swoim plikiem,
		- restartuje system.
		Rootkit na poziomie jądra to program który:
			- wstrzykuje się w jądro systemu,
			- przechwytuje wywołania systemowe (syscalls),
			- filtruje odpowiedzi - usuwa te, które pasują do jego konfiguracji, np PID, nazwa pliku, port, IP. 
			Należy pamiętać, że programy takie jak `ps`, `ls`, `netstat` działają w przestrzeni użytkownika. Żeby uzyskać informacje o procesach, połączeniach sieciowych lub plikach, programy te pytają jądro systemu (kernel) za pomocą wywołań systemowych (syscalls). Ponieważ rootkit działa wewnątrz jądra może przechwycić i modyfikować odpowiedzi. Dlatego też:
				- gdy `ps` pyta o procesy, rootkit usuwa z odpowiedzi procesy, których PID lub nazwa znajdują się w jego konfiguracji,
				- gdy `ls` pyta o pliki, rootkit usuwa z odpowiedzi, te które ma zaprogramowane,
				- analogiczna sytuacja występuje dla pozostałych zapytań systemowych takich jak `netstat`, `who`, `lsmod`. Nawet specjalne narzędzia stworzone do wykrywania rootkitów takie jak `rkhunter` czy `chkrootkit` mogą nie zadziałać, bo polegają na tych samych wywołaniach systemowych. 
		Zagrożenie jest o tyle niebezpieczne, że proces podmiany jądra nie wymaga fizycznego dostępu, działa również przez SSH, jedynym warunkiem jest dostęp zdalny do roota. 
		- jak wykryć:
			- dobrą praktyką jest zachowanie sumy kontrolnej zaraz po instalacji systemu, dzięki czemu można później sprawdzić w każdej chwili czy suma kontrolna zgadza się z tym co było na początku. Jeśli wynikiem jest "OK" jądro nie zostało podmienione, wynik "FAIL" oznacza podmianę jądra. 
				- `md5sum /boot/vmlinuz-$(uname -r) > /root/kernel_md5.txt` - zapis sumy kontrolnej po instalacji systemu,
				- `md5sum -c /root/kernel_md5.txt` - porównanie sum kontrolnych obecnej i początkowej. 
			- innym sposobem jest porównanie z pakietem (Debian/Ubuntu):
				- `dpkg -S /boot/vmlinuz-$(uname -r)` - sprawdzenie z jakiego pakietu pochodzi jądro,
				- `sudo debsums $(dpkg -S /boot/vmlinuz-$(uname -r) | cut -d: -f1)` - weryfikacja sumy kontrolnej pakietu. Polecenie `debsums` porównuje sumę kontrolną pliku jądra z właściwą wartością zapisaną w bazie danych pakietu i zwraca "OK" jeśli jest zgodna lub "FAIL" jeśli sumy się nie zgadzają.
				- można także sprawdzić datę modyfikacji pliku jądra: `ls -la /boot/vmlinuz-$(uname -r)`, jeśli data jest inna niż data instalacji - jądro zostało zmienione. 
		- jak naprawić: 
			- `sudo apt install --reinstall linux-image-$(uname -r)` - przeinstalowanie pakietu jądra (Debian/Ubuntu),
			- po reinstalacji należy zrestartować system.
			To jednak nie zawsze jest w stanie usunąć rootkita jądra, są sytuacje, w których jedynym działającym rozwiązaniem jest przeinstalowanie całego systemu lub przywrócenie backupa. 
- `systemd`:
	- zagrożenie: atakujący może dodać usługę, która uruchamia skrypt np. reverse shell.  
	Usługa startuje automatycznie przy każdym starcie systemu i działa jako root, a jeśli usługa padnie, systemd restartuje ją. 
	- jak wykryć:
		- `systemctl list-unit-files --type=service --state=enabled` - lista wszystkich usług uruchamianych przy starcie, 
		- `systemctl list-units --type=service --state=running` - lista aktywnych usług,
		- `systemctl cat nazwa_usługi.service` - czytanie szczegółów usługi, 
		- `systemctl list-unit-files --type=service --state=enabled | grep -E "backup|update|fix|temp|cache"` - sprawdzenie, czy nie ma usług z dziwnymi nazwami (np. backup.service, system-update.service, network-fix.service, cron-helper.service).
		- jeśli usługa wygląda podejrzanie, należy sprawdzić:
			- `systemctl cat nazwa.service | grep ExecStart` - gdzie jest skrypt. Jeśli ExecStart wskazuje na katalog `/tmp`, `/opt`, `/usr/local/bin` lub `/lib/systemd/` to czerwona flaga, 
			- `cat /usr/local/bin/backdoor.sh` - czy skrypt istnieje i co zawiera. Należy zwrócić uwagę czy skrypt zawiera `nc -e /bin/bash`, `bash -i >& /dev/tcp/`, `python -c import socket...` - to oznacza reverse shell,
			- `ls -la /usr/local/bin/backdoor.sh` - kto jest właścicielem skryptu. Jeśli skrypt ma datę modyfikacji z czasu po ataku lub/i właścicielem skryptu jest użytkownik inny niż root (np. www-data) to jest alarmujące. 
	- jak naprawić (remediacja):
		- `systemctl stop nazwa.usługi` - zatrzymać usługę,
		- `systemctl disable nazwa.usługi` - wyłączyć usługę by nie startowała przy boot,
		- `rm /etc/systemd/system/nazwa.usługi` - usunąć plik usługi, 
		- `rm /usr/local/bin/backdoor.sh` - usunąć skrypt, 
		- ` systemctl daemon-reload` - przeładować systemd (odświeżyć listę usług). 
		- sprawdzić, czy  nie ma innych backdoorów (cron, inne usługi, GRUB, initramfs). 

## Wnioski bezpieczeństwa 
Wszystkie opisane powyżej ataki wymagają uprawnień roota, by mogły skończyć się powodzeniem. Więc jeśli któryś z nich zakończył się powodzeniem, oznacza to, że atakujący uzyskał dostęp do root, dlatego poza powyższymi rozwiązaniami należy podjąć także kroki, które zabezpieczą na nowo konto root. Bez tego zabiegu wszystkie działania naprawcze nie przyniosą skutku, ponieważ najważniejsze jest odcięcie atakującego od możliwości, które daje dostęp do root. Priorytetem jest:
- ochrona konta root - silne hasło, wyłączenie logowania przez SSH (lub zezwolenie na logowanie tylko za pomocą klucza), monitorowanie logów, 
- monitorowanie zaufanych kluczy SSH - `/root/.ssh/authorized_keys` oraz `/home/*/.ssh/authorized_keys`,
- regularna zmiana hasła roota, zwłaszcza po podejrzanych zdarzeniach, 
- audyt dostępu roota - `sudo -l`, `journalctl _UID=0`, `grep "COMMAND" /var/log/auth.log`.