# ps_top_htop

## Cel
Zrozumienie sposobów monitorowania procesów w systemie. 

## ps
Wypisuje listę procesów w danym momencie - to szybki podgląd, bez odświeżania.   
Podstawowe opcje:
- `ps aux` - wyświetla wszystkie procesy, niezależnie od terminala, w czytelnym formacie. To zwykle pierwszy krok, by zobaczyć co aktualnie działa.   
- `ps auxf` - wyświetla drzewo procesów (kto kogo uruchomił) - za pomocą tego polecenia można sprawdzić czy jakiś podejrzany proces ma podejrzanego rodzica, 
- `ps -eo pid,ppid,user,cmd,%cpu,%mem --sort=-%cpu` - format niestandardowy, sortowanie po CPU (malejąco). Umożliwia znalezienie 'winowajcy' gdy system działa wolno,
- `ps -u root` - wyświetla tyko procesy użytkownika root, czy ma uruchomiony jako dziecko jakiś podejrzany proces, 
- `ps --ppid 2` - wyświetla procesy, których rodzic ma PID 2 (kworker). Pomaga w sprawdzeniu, czy ktoś nie podpiął się pod fałszywy `kworker`.   
Przykład:
- `ps aux | grep -E "nc|bash|python|perl|/tmp"` - polecenie to szuka podejrzanych procesów (revers shelle często używają `nc`, `bash` z podejrzanym rodzicem. skrypty z `/tmp`. 

## Czym jest kworker
Kworker to proces systemowy Linuxa, który wykonuje zadania w tle na zlecenie jądra (np. obsługa dysków, sieci, sterowników). Jest też dość częstym wektorem ataków - atakujący podpina swój proces, z nazwą kworker, żeby działać z ukrycia. 
- Jak rozpoznać czy kwoker jest prawdziwy czy podrobiony: 
	- prawdziwy kworker: należy do roota, mieszka w `/usr/bin` lub `/`, nie obciąża CPU nieustannie, nie tworzy plików w `/tmp`,
	- fałszywy kworker (malware): może należeć do innego użytkownika (np. nobody, www-data), mieszka w `/tmp` lub `/home`. zużywa CPU, otwiera podejrzane pliki. 
- jak rozpoznać (polecenia):
	- `ps aux | grep kworker |grep -v root` - sprawdzenie właściciela, powinien być to root, jeśli to polecenie wyświetli cokolwiek - czerwona flaga,  
	- `for pid in $(pgrep -f kworker); do echo "PID $pid:"; ls -la /proc/$pid/exe 2>/dev/null; done` - sprawdzenie skąd działają kworkery, ścieżka powinna być systemowa, jeśli ścieżka wskazuje na `/tmp`, `/home` - czerwona flaga,
	- `top -b -n 1 | grep kworker | awk '{print$9}'` - wyświetla % CPU dla każdego kworkera. jeśli ciągnie powyżej 50% - jest to podejrzane.
		
## top
To program do monitorowania procesów w czasie rzeczywistym. Pokazuje, co się dzieje w systemie "na żywo" - które procesy jedzą CPU, pamięć, itp. Poruszanie się po programie:
- `h` - pomoc, 
- `q` - wyjście,
- `shift + P` - sortowanie po CPU,
- `shift + M` - sortowanie po pamięci,
- `k` - zabija proces (należy podać PID),
- `u` - pokazuje procesy tylko dla jednego użytkownika (należy wpisać nazwę użytkownika),
- `l` - włącza/wyłącza wiersz obciążenia, 
- `t` - włącza/wyłącza wiersz zadań (tasks),
- `m` - włącza/wyłącza wiersz pamięci,
- `1` - rozwiń każdy rdzeń CPU osobno.   

## htop
To jest nowsza, bardziej czytelna wersja top (czasem trzeba doinstalować), ma więcej opcji i koloruje wyniki. Poruszanie się w `htop`:
- `f6` (lub kliknięcie nagłówka kolumny) - sortowanie (CPU, MEM, PID, TIME),
- `F3` - wyszukiwanie procesu po nazwie, 
- `F4` - filtrowanie (pokaż tylko procesy zawierające tekst),
- `F5` - widok drzewa (kto kogo uruchomił),
- `F9` - zabij proces (wybierz PID, potem sygnał 15 TERM lub 9 KILL),
- `F10` / `q` - wyjście,
- `strzałki` - poruszanie się po liście. 
Z punktu widzenia SOC `htop` jest lepszy ponieważ:
- koloruje procesy, co ułatwia zauważenie podejrzanych, 
- widok drzewa (F5) pokazuje relacje rodzic-dziecko,
- można wyszukiwać i filtrować. 

## Na co zwracać uwagę podczas monitorowania procesów
- lokalizacja - ` /tmp/plik`, `/dev/shm/x`, `/home/user/.cache/backdoor` - procesy systemowe nie uruchamiają się z tych katalogów. Należy sprawdzić proces za pomocą polecenia `lsof -p PID`, a następnie zabić go, 
- nazwa - `sshd` ale nie w `/usr/sbin`, `[kworker]` z przecinkami - udaje znany proces, to może być rootkit. Należy sprawdzić proces za pomocą polecenia `ls -la /proc/PID/exe`, które pomoże ustalić prawdziwą ścieżkę, 
- użytkownik - `nobody` lub `www-data` uruchamia `nc`, `bash` `python` - konto usług nie powinno robić takich rzeczy, należy sprawdzić czy to nie jest backdoor, 
- rodzic (PPID) - `bash` -> `nc` (jako root) lub `cron` -> `python` -> `nc` - podejrzany łańcuch zdarzeń, (np. czemu cron uruchamia skrypt z `/tmp`). Należy sprawdzić crontab, systemd,
- CPU - proces z 300% CPU - może być minerem kryptowalut lub pętlą DoS. Należy zabić proces, 
- sieć - proces z otwartym portem 4444, 1337, 31337 - reverse shell. Należy sprawdzić za pomocą : `lsof -i -P -n` lub `netstat -tunlp`.

## Polecenia przydatne w głębszym przyjrzeniu się podejrzanym procesom
- `ls -la /proc/PID/exe` - pokazuje prawdziwą ścieżkę do pliku (nawet jeśli proces się ukrywa),
- `cat /proc/PID/cmdline` - pokazuje z jakimi argumentami został uruchomiony proces,
- `lsof -p PID` - jakie pliki, sockety, połączenia sieciowe proces ma otwarte,
- `strace -p PID` - jakie wywołania systemowe wywołuje proces. 

## Case study

Sytuacja:  
-Serwer produkcyjny Ubuntu działa wolno. W godzinach nocnych (3:00-4:00) skokowo rośnie zużycie CPU (do 80-90%). Poza tymi godzinami CPU jest na poziomie 5-10%.  
-Wykonane czynności:
	- `top` z sortowaniem po CPU daje wynik:  
PID     USER       PR   NI   VIRT      RES     SHR   S  %CPU   %MEM  TIME+ COMMAND    
1234    root       20   0    250000    50000   2000  S  85.5   2.5   120:30.12 [kworker/0:1]  
5678    www-data   20   0    120000    30000   1500  S   5.0   1.5   10:20.15 php-fpm   
9012    root       20   0     50000    10000   2000  S   1.0   0.5   5:10.22 systemd  
	- polecenie ls -la /proc/1234/exe:  
lrwxrwxrwx 1 root root 0 ... /proc/1234/exe -> /tmp/.hidden/update  
	- polecenie cat /proc/1234/cmdline: pusty, nic nie pokazuje, 
	- polecenie lsof -p 1234:   
COMMAND  PID   USER  FD   TYPE  DEVICE SIZE/OFF  NODE  NAME  
update   1234  root  cwd  DIR   8,1       4096   2     /  
update   1234  root  txt  REG   8,1     123456   10    /tmp/.hidden/update (deleted)  
update   1234  root   3u  IPv4  56789      0t0   TCP   10.0.0.25:4444->185.130.5.253:12345 (ESTABLISHED)  
	- sprawdzenie cron roota: sudo crontab -l:  
0 3 * * * /tmp/.hidden/update  

Analiza:
- Kworker nie jest prawdziwy, ponieważ:
	- uruchamia się z katalogu /tmp, co nie powinno mieć miejsca. Prawdziwe kworkery startują z katalogów systemowych,
	- zajmuje bardzo dużo CPU, prawdziwe kworkery nie potrzebują dużo CPU,
	- ma otwarte połączenie reverse shell, 
	- jest ustawiony w cron na nietypowe godziny (noce, poza godzinami pracy).
- atakujący w jakiś sposób uzyskał uprawnienia root, umieścił skrypt w katalogu znajdującym się w /`tmp`  z pełnymi prawami dla wszystkich, po czym uruchomił proces, 
- usunięty plik miał za zadanie utrzymać stałe połączenie zdalne (ESTABLISHED). Plik mógł zostać usunięty przez samego atakującego, lub kogoś z IR.  

Zalecanie działania:  
- należy sprawdzić kto utworzył proces za pomocą polecenia `ps aux`, jeśli właścicielem jest root, należy sprawdzić w `/var/log/auth.log` kto logował się na roota przed atakiem. Polecenie `sudo -l` pomoże ustalić listę podejrzanych kont,
- należy zatrzymać proces poleceniem `kill -9 1234`, 
- sprawdzić, kto jest właścicielem pliku znajdującego się w `/tmp/.hidden/` - ponieważ plik został usunięty jedynym sposobem jest sprawdzenie tego w audit, pod warunkiem, że został skonfigurowany przed atakiem,
- w zależności od narzędzi skonfigurowanych w systemie, sprawdzić kiedy plik został utworzony:
	- za pomocą logów `/var/log/auth.log`, `/var/log/syslog`,
	- za pomocą audit jeśli był skonfigurowany, 
	- za pomocą historii powłoki poszukując poleceń, które mogły umieścić plik w systemie (wget, curl, nano, touch, chmod),	
- usunąć pliki z `/tmp` poleceniem `rm /tmp/.hidden/`,
- usunąć proces z cron, za pomocą `sudo crontab -e`, następnie wybrać wpis, otworzyć i ręcznie usunąć wpis `0 3 * * * /tmp/.hidden/update`,
- sprawdzić logowania oraz działania wykonane ze skompromitowanego konta od czasu kompromitacji. Jeśli w systemie jest skonfigurowany auditd i sesja ma ustawiony loginuid, wystarczy użyć polecenia `sudo ausearch -ua UID -i`,
- sprawdzić wszystkie katalogi pod kątem innych złośliwych skryptów (/tmp, /var/tmp, /dev/shm),
- zmienić hasło dla tego konta i prewencyjnie dla roota, 
- sprawdzić pozostałe newralgiczne pliki systemu pod kątem modyfikacji od czasu skompromitowania konta (`/etc/sudoers`, `/etc/sudoers.d/`, `/etc/pam.d/`, `/etc/pam/`, systemd, `/etc/passwd`, `/etc/group`, `/home/*/.ssh/authorized_keys`, `/root/.ssh/authorized_keys` `crontab` wszystkich użytkowników, `/etc/ssh/sshd_config`, itp)

