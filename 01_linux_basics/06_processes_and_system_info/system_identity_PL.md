# system_identity

## Cel
Zrozumienie, jakie podstawowe informacje o systemie można odczytać i dlaczego ich zmiana może wskazywać na atak. 

## Podstawowe polecenia

### Jądro (uname -a) 
- polecenie to pokazuje informacje o jądrze, architekturze, nazwie hosta, dacie kompilacji jądra. Przykład: `Linux hostname 5.15.0-91-generic #101-Ubuntu SMP x86_64 GNU/Linux`
- zagrożenie - podmiana jądra (rootkit). Atakujący podmienia plik jądra (`/boot/vmlinuz-...`) na swoją wersję, która zawiera rootkita. 
	- Rootkit może:
		- ukrywać procesy (nie będą widoczne w `ps`, `top`),
		- ukrywać pliki (nie będą widoczne w `ls`, `find`),
		- ukrywać połączenia sieciowe (nie będą widoczne w `netstat`, `ss`),
		- ukrywać swoje moduły (`lsmod` ich nie pokaże).
	- dlaczego to niebezpieczne:
		- rootkit jest trudny do wykrycia, standardowe narzędzia (nawet `rhhunter`, `chkrootkit`) mogą go nie wykryć,
		- atakujący może działać w ukryciu, czytać dane, logować hasła, wysyłać ruch.
- sprawdzenie, z jakiego pakietu pochodzi aktualne jądro (Debian/Ubintu):
	- `dpkg -l /boot/vmlinuz-$(uname -r)`,
- sprawdzenie, czy suma kontrolna się zgadza:
	- `sudo debsums linux-image-$(uname -r) 2>/dev/null | grep vmlinuz` - jeśli wynik jest `OK` znaczy, że jądro nie zostało zmienione, jeśli `FAILED` jądro zostało zmienione - czerwona flaga,
- porównanie daty pliku jądra z datą instalacji systemu:
	- `ls -la /boot/vmlinuz-$(uname -r)` - jeśli data jest inna niż instalacji systemu - możliwa podmiana.
- jak to naprawić:
	- należy przeinstalować pakiet jądra: `sudo apt install --reinstall linux-image-$(uname -r)`,
	- jeśli rootkit jest za głęboki jedynym ratunkiem jest reinstalacja systemu lub przywrócenia backupu. 

### Restart systemu (uptime, last reboot)
- `uptime` - pokazuje jak długo system działa. Przykład: `10:30:00 up 5 min, 2 users, load average: 0.00, 0.02, 0.05`
- `last reboot` - pokazuje historię restartów (data, godzina), 
- jak sprawdzić:
	- `uptime`,
	- `last reboot | head -5`.
- zagrożenia - atakujący może zrestartować system po:
	- podmianie jądra - nowe jądro musi się załadować,
	- modyfikacji GRUB (`init=/bin/bash`) - restart jest konieczny, 
	- instalacji backdoora, wymagającego restartu.
- dlaczego jest niebezpieczne:
	- po restarcie system traci pamięć, znikają logi, procesy,
	- atakujący może ukryć swój proces, który działał przed restartem, 
	- analityk SOC, widzi, że system działa krótko - należy zadać pytanie dlaczego?
- wykrywanie nieplanowanych restartów:
	- `last reboot` - historia restartów. Naturalne restarty zwykle mają miejsce w godzinach pracy admina. Restarty np. o 3 w nocy lub weekendy budzą podejrzenia, 
	- `grep -E "reboot|shutdown" /var/log/auth.log` - sprawdzenie, czy ktoś nie uruchomił `reboot`, lub `shutdown`,
	- `grep "Kernel panic" /var/log/syslog` - sprawdzenie, czy system nie został zrestartowany z powodu błędu.
- zalecane działania po nieplanowanym restarcie:
	- należy sprawdzić, czy jądro nie zostało podmienione, 
	- sprawdzić, czy GRUB nie został zmodyfikowany (`init=/bin/bash`),
	- sprawdzić, logi sprzed restartów w archiwach `/var/log/`.

### Nazwa hosta (hostname, /etc/hostname)
- `hostname` - pokazuje aktualną nazwę hosta, 
- jak sprawdzić: 
	- `hostname`,
	- `cat /etc/hostname` - prawdziwa poprawna nazwa.
- zagrożenie - atakujący zmienia nazwę, aby utrudnić śledztwo,
- dlaczego to niebezpieczne:
	- logi (`auth.log`, `syslog`) zapisują nazwę hosta odczytaną w momencie zdarzenia, 
	- jeśli atakujący zmieni nazwę przed atakiem, logi będą pokazywać fałszywą nazwę, 
	- `grep` po prawdziwej nazwie nie znajdzie wyników z okresu ataku, 
	- SIEM może traktować prawdziwą i fałszywą nazwę jako dwa oddzielne źródła - alerty nie zostaną skorelowane. 
- jak wykryć: 
	- `grep "hostname changed from" /var/log/auth.log` - sprawdzenie, czy nazwa była zmieniana,
	- `hostname`, `cat /etc/hostname` - należy porównać nazwy wyświetlone przez te dwa polecenia, jeśli się różnią ktoś zmienił nazwę tymczasowo,
	- `grep -E "localhost|inna_nazwa" /var/log/syslog` - sprawdzenie logów systemowych pod kątem wpisów z inną nazwą hosta. 
- jak naprawić:
	- należy przywrócić prawidłową nazwę: `sudo hostname prawidłowa_nazwa`,
	- zmienić w `/etc/hostname` (jeśli zachodzi potrzeba stałej zmiany).

### Adres IP (hostname -i, /etc/hosts)
- Polecenie `hostname -i` pokazuje adres IP powiązany z nazwą hosta (z pliku `/etc/hosts` lub DNS).
- jak sprawdzić prawdziwy adres IP:
	- `ip addr show`,
- zagrożenie - modyfikacja `/etc/hosts`. Atakujący dodaje wpis w pliku `/etc/hosts`, który przekierowuje ruch. Przykład podejrzanego wpisu: `185.130.5.253 google.com`.
- dlaczego to niebezpieczne:
	- serwer łącząc się z `google.com` trafi na IP atakującego (może pobrać malware),
	- skrypt sprawdzający `hostname -i` może dostać inne IP niż oczekiwane. 
- jak wykryć:
	- `cat /etc/hosts` - należy szukać wpisów, które:
		- przypisują znane domeny takie jak np. google.com, github.com do dziwnych IP,
		- przypisują nazwę hosta do innego IP niż rzeczywiste.
- jak naprawić:
	- należy usunąć podejrzane wpisy z `/etc/hosts`,
	- lub przywrócić domyślny plik z backup. 

### Dystrybucja (/etc/os-release)
- pokazuje rodzaj dystrybucji i wersję,
- jak sprawdzić:
	- `cat /etc/os-release`,
	- `lsb_release -a` - jeśli dostępne. 
- zagrożenie - zmiana dystrybucji (fałszywy `os-release`). Atakujący podmienia plik, żeby system wyglądał na inny. 
- dlaczego to niebezpieczne:
	- skrypty automatyzujące sprawdzają (np. do aktualizacji) `os-release` i wybierają odpowiednie polecenia (apt, yum, dnf). Jeżeli dystrybucja jest fałszywa, skrypt może wykonać błędne polecenia. 
	- narzędzia do audytu (np. lynis) mogą błędnie ocenić podatności, 
	- analityk SOC może szukać podatności dla złej dystrybucji.
- jak wykryć:
	- należy porównać dystrybucję i wersję z wynikiem polecenia `uname -a`,
	- sprawdzić datę modyfikacji pliku: `ls -la /etc/os-release`, jeśli data jest świeża jest to podejrzane,
	- porównać z domyślnym plikiem dla danej dystrybucji (wzorzec można znaleźć w internecie).
- jak naprawić:
	- przywrócić oryginalny plik z backupu.