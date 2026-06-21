## /etc/sudoers

## Cel laboratorium
Celem tego laboratorium było zapoznanie się jak działają zapisy w pliku `/etc/sudoers` oraz jaki to ma wpływ na uprawnienia użytkowników.

### Czym jest /etc/sudoers
Sudoers to plik konfiguracyjny programu sudo, który kontroluje:
- kto może tymczasowo przyjąć inny UID/GID,
- jako jaki użytkownik, 
- w jakim zakresie poleceń,
- na jakich warunkach (hasło, brak hasła, ograniczenia środowiska).  
Czym sudoers nie jest:
- nie nadaje praw systemowych,
- nie zmienia właścicieli plików, 
- nie nadaje magicznych uprawnień grupie root,
- nie jest mechanizmem kontroli dostępu do plików.  
Plik sudoers określa, czy proces uruchomiony przez sudo może wystartować z innym UID/GID oraz na jakich warunkach.Nie nadaje uprawnień - umożliwia uruchomienie procesu z uprawnieniami innego użytkownika. Plik główny `/etc/sudoers` może zawierać dyrektywę #includedir, która wczytuje dodatkowe reguły z katalogu `/etc/sudoers.d/`.

## Struktura wpisów
Struktura wpisów w pliku `/etc/sudoers` ma postać:   
	`USER HOST=(RUNAS_USER:RUNAS_GROUP) COMMAND`  
- `HOST` - ma znaczenie w środowiskach LDAP/NIS, na zwykłym systemie jest zwykle ALL,
- `RUNAS` - jako kto ma zostać uruchomiony proces, może to być:
	- (root),
	- (backup),
	- (www-data),
	- (ALL),
	- (ALL:ALL).  
	Należy pamiętać, że to nie zmienia użytkownika logowania tylko określa UID/GID procesu, który zostaje uruchomiony.    
	UWAGA: Jeśli we wpisie sudoers nie zostanie podany blok (RUNAS), domyślnie przyjmowany jest użytkownik root. Celem wyświetlenia listy poleceń, które aktualny użytkownik może wykonać przy użyciu sudo zgodnie z obowiązującą konfiguracją sudoers - stosuje się polecenie:
	- `sudo -l`
	to: 
	- pokazuje dopasowane reguły,
	- informuje, czy wymagane będzie hasło,
	- uwzględnia wpisy z `/etc/sudoers` oraz `/etc/sudoers.d/`.  
	Możliwe jest również sprawdzenie uprawnień innego użytkownika (jako root):  
		- `sudo -l -U username`  
- `COMMAND` - co dokładnie wolno uruchomić  - sudo nie ufa PATH przy wyszukiwaniu komendy, dlatego zaleca się pełne ścieżki, dodatkowo `secure_path` może nadpisywać PATH użytkownika. Przykład:
	- `/usr/bin/apt`
	- `/bin/systemctl`

## Jak to działa   
O sudo prosi:  
- konkretny użytkownik,  
- %grupa,
- ALL.  
Na przykład `%sudo ALL=(ALL:ALL) ALL` oznacza, że:
- każdy w grupie sudo,
- może z każdego hosta,
- jako dowolny user i dowolna grupa,
- wykonać dowolne polecenie.  
Kolejność przetwarzania:
- sudo czyta `/etc/sudoers`,
- wczytuje #includedir `/etc/sudoers.d`,
- reguły czytane są z góry do dołu, a jeśli są dwa takie same wpisy:    
	`user ALL=(ALL) ALL`    
	`user ALL=(ALL) NOPASSWD: ALL`    
	to ten poniższy wpis nadpisze ten, który znajduje się nad nim.   
	UWAGA: 'Nadpisanie' w sudoers występuje tylko wtedy gdy:  
		- USER pasuje,
		- HOST pasuje,
		- RUNAS pasuje,
		- COMMAND pasuje,  
	Sudo nie wybiera jednej reguły, sudo zbiera wszystkie pasujące wpisy, a potem dla opcji typu NOPASSWD decyduje ostatni. ALL oznacza 'każda komenda', więc może dopasować się raz z bardziej szczegółową regułą; w przypadku konfliktu decyduje kolejność w pliku (ostatnia wygrywa dla PASSWD/NOPASSWD).  
Matchowanie ALL: znaczy pasuje w danym polu, ale polecenia nadal musi istnieć i być wykonalne. `ALL=(ALL) ALL` - oznacza pełną eskalację. 

## Defaults
Są to ustawienia sterujące zachowaniem programu sudo podczas wykonywania polecenia z podwyższonymi uprawnieniami.Należy pamiętać, że sudo to nie jest zwykłe polecenie, to jest program, który pozwala uruchomić inny program z UID rota (lub innego użytkownika), zgodnie z regułami w sudoers. Ustawienia Defaults nie nadają praw, tylko kontrolują:
- jak sudo uwierzytelnia użytkownika,
- jakie zmienne środowiskowe zostaną zachowane lub wyczyszczone,
- jaki PATH zostanie ustawiony,
- czy wymagany jest terminal, 
- jak długo ważna jest sesja uwierzytelnienia.   
Reguły te nie wykonują się przy logowaniu, tylko przed uruchomieniem polecenia z podwyższonymi uprawnieniami, przygotowują wszystko pod ten proces.  
Schemat wygląda tak:  
- użytkownik wpisuje: sudo apt update,
- sudo wczytuje sudoers i Defaults,
- sprawdza, czy użytkownik może wykonać polecenie, 
- jeśli tak - stosuje Defaults (env_reset, secure_path, itd),
- potem zmienia UID i exec().  
Czyli:  
Defaults wpływa na proces, który  będzie uruchomiony przez sudo. Przykłady reguł Defaults:
- Defaults env_reset - to jest jedno z najważniejszych ustawień.
	- czyści (resetuje) większość zmiennych środowiskowych użytkownika, zanim uruchomi program jako root, to jest ważne, ponieważ zmienne środowiskowe mogą wpływać na działanie programu, np:
		- LD_PRELOAD,
		- PATH,
		- PYTHONPATH,
		- EDITOR.  
	Gdyby sudo nie czyściło środowiska, użytkownik mógłby:
		- ustawić złośliwe LD_PRELOAD,
		- podmienić bibliotekę,
		- uruchomić proces root z manipulowanym środowiskiem.   
	env_reset zapobiega temu. 
- Defaults secure_path - nadpisuje zmienną PATH dla procesu uruchomionego jako root. Czyli nawet jeśli użytkownik ma:    
	`PATH=/home/usr/bin:/usr/bin`     
	to proces root dostanie:    
 	`PATH=/usr/sbin:/usr/bin:/sbin:/bin`    
 	zdefiniowane w sudoers.   
 	To zabezpiecza przed sytuacją:  
 	- użytkownik tworzy fałszywy program ls w swoim katalogu, 
 	- zmienia PATH,
 	- sudo uruchamia niewłaściwy binarny plik,  
- Defaults requiretty - wymusza, aby sudo było wywołane z prawdziwego terminala, czyli:
	- nie z crona,
	- nie z procesu bez TTY,
	- nie z niektórych skryptów.  
	To było kiedyś popularne zabezpieczenie na serwerach. 

### Wnioski bezpieczeństwa
Na co trzeba zwrócić uwagę:
- Kto ma pełny dostęp, w pliku /etc/sudoers widać jakie prawa mają grupy root i sudo, sprawdzić w `/etc/group` kto należy do tych grup i czy na pewno powinien tam należeć, 
- Kto ma NOPASSWD,
- Czy są nietypowe wpisy w pliku `/etc/sudoers`,
- Czy są nietypowe, nieznane pliki w katalogu `/etc/sudoers.d/`

### Potencjalne zagrożenia
- Każdy użytkownik z NOPASSWD to potencjalny wektor ataku, 
- Nietypowe polecenia pozwalające na zapis do systemowych katalogów (`/etc`, `/root`, `/var`) powinny być dokładnie sprawdzone, czy rzeczywiście powinny tam być, 
- Jeśli użytkownik jest w grupie sudo, dla której konfiguracja pliku `/etc/sudoers` nadaje pełne uprawnienia, nie ma możliwości zablokowania mu czegokolwiek, dlatego jeśli użytkownik ma mieć tylko niektóre prawa roota należy usunąć go z grupy sudo i dodać stosowny plik w `/etc/sudoers.d/` by nadać mu indywidualne uprawnienia,
- Chcąc zabronić użytkownikowi dostępu  do `/bin/bash` ze względów bezpieczeństwa, nie można zrobić tego dodając wpis w `/etc/sudoers.d/plikusera` `user ALL = (ALL) ALL !/bin/bash`, ponieważ można to obejść korzystając z innych powłok niż bash, np użycie podpowłoki zsh,
- Nie zadziała również zapis `user ALL =(ALL) ALL !/bin/bash, !/bin/sh, !/bin/zsh`, ponieważ nadal można to obejść na kilka sposobów:
	- `sudo -s` - uruchamia powłokę jako root ale zachowuje część środowiska użytkownika - to jest 'shell jako root',
	- `sudo -i` - symuluje pełne logowanie jako root - ładuje profil roota, zmienia HOME na /root, zmienia środowisko. To jest 'login shell root', 
	- `env` - pozwala użytkownikowi ustawiać lub przeglądać zmienne środowiskowe, ale przy sudo potencjalnie może służyć do eskalacji, jeśli Defaults env_reset nie jest ustawione.
	- Powyższe metody działają bo w sudoers jest najczęściej:
		- `%sudo ALL=(ALL:ALL) ALL`,
		- czyli wolno uruchomić każde polecenie, 
		- a `/bin/bash` jest poleceniem.   
		Więc jeśli możesz uruchomić dowolne polecenie jako root, możesz uruchomić powłokę jako root. Dlatego czasem łatwiej jest zapisać w sudoers co wolno danemu użytkownikowi/grupie niż czego mu nie wolno. Bo problem nie jest w bashu, tylko w tym że pozwalasz na uruchomienie programu z UID 0.
- Najskuteczniejszym sposobem zabezpieczenia jest zapis w osobnym pliku dla danego użytkownika w `/etc/sudoers.d/`, gdzie można przykładowo nadać uprawnienia root użytkownikowi dla poleceń apt update i apt upgrade za pomocą zapisu w pliku : `user ALL=(root) /usr/bin/apt update, /usr/bin/apt upgrade`.
- Tworzenie oddzielnych plików w `/etc/sudoers.d/` umożliwia nadawanie granularnych uprawnień, ograniczając ryzyko pełnego dostępu do roota. 
- Każdy nowy wpis w sudoers lub sudoers.d powinien być weryfikowany pod kątem nietypowych poleceń, zwłaszcza zapisów umożliwiających zapis do systemowych katalogów (`/etc`, `/root`, `/var`).

## Analiza wpisów w pliku /etc/sudoers  (Case Study)

### Case study 1  
```
 Konfiguracja:  
 	Defaults env_reset  
 	Defaults:user1 !requiretty  
 	user1 ALL=(root) /usr/bin/apt update, /usr/bin/apt upgrade  

 Sytuacja:  
 - user1 loguje się zdalnie przez ssh (bez terminala interaktywnego)
 - ma ustawioną zmienną środowiskową LD_PRELOAD=/home/user1/malicious.so
 - próbuje uruchomić:
 	- sudo /usr/bin/apt update
 	- sudo /usr/bin/id
 	- sudo -i  
```
 Analiza:  
 - konfiguracja ma ustawione resetowanie wszystkich zmiennych środowiskowych przed użyciem polecenia z podwyższonymi uprawnieniami, jednak jednocześnie '!' przy requiretty pozwala na korzystanie z poleceń sudo nie tylko przez TTY,
 - user1 ma w sudoers zapis pozwalający na korzystanie z poleceń apt update i apt upgrade z programem sudo, nie ma dostępu do polecenia id, 
 - niezależnie od sposobu logowania user1 ma dostęp do zapisanych mu w sudoers pozwoleń na użycie poleceń,
 - ustawiona ścieżka środowiskowa przez użytkownika nie zadziała z sudo, ponieważ default env_reset ją zresetuje przed uruchomieniem polecenia z sudo,
 - user1 może wykonać polecenie `apt update` z uprawnieniami sudo,
 - uruchomienie `sudo -i` oraz `sudo /usr/bin/id` skończą się niepowodzeniem, ponieważ polecenia nie są dozwolone w sudoers.   

Sugerowane działania:  
- konfiguracja jest bezpieczna, nie wymaga działań,
- opcjonalnie można rozważyć czy nie warto ograniczyć sudo do prawdziwego TTY.

### Case study 2
 ```
 Konfiguracja sudoers:   
     - %dev ALL=(ALL) ALL  
     - %backup ALL=(ALL) /usr/bin/rsync  
 Grupy w systemie:  
     - dev:x:1050:user2
     - backup:x:1050:user2   
 Sytuacja:
     - user2 ma primary group dev i secondary backup
     - próbuje uruchomić:  
     	- sudo /usr/bin/rsync  
     	- sudo /bin/ls  
 ```
 Analiza:  
 - Występuje tu problem z duplikacją GID grupy i tym trzeba się niezwłocznie zająć, 
 - sudoers działa na nazwach przestrzeni userspace, nie po numerze GID, a więc user2 dostanie pełne uprawnienia stosowane do zapisu w grupie dev i w grupie backup. Więc ma pełne prawa systemowe. 
 - jeśli grupa dev zostanie usunięta, użytkownik będzie miał tylko uprawnienia grupy backup, więc nie będzie mógł wykonać polecenia ls.  

 Sugerowane działania:
 - należy przede wszystkim rozwiązać kolizję GID grup, zmieniając jeden z nich na niekolidujący z żadną inna grupą numer i uporządkować własności. 
 - sprawdzić w plikach historycznych, jaka grupa była pierwotna dla user2, kto i kiedy zmienił numer GID jednej z tych grup (dev i backup, zmiana zapewne wykonana ręcznie, ponieważ stosowne polecenia zarządzające grupami GID).


 ### Case study 3 
```
 Konfiguracja sudoers:  
 	Defaults secure_path="/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin"   
 	user3 ALL=(root) /usr/bin/systemctl restart nginx   
 	user3 ALL=(www-data) /usr/bin/touch /var/www/html/testfile   
 Sytuacja:  
 	user3 próbuje uruchomić:  
 	- sudo /usr/bin/systemctl restart nginx
 	- sudo /usr/bin/systemctl restart apache2
 	- sudo -i 
 	- sudo -u www-data /usr/bin/touch /var/www/html/testfile
 	- sudo -u www-data /usr/bin/touch /root/hackfile
```
 Analiza:
 - `sudo /usr/bin/systemctl restart nginx`, `sudo -u www-data /usr/bin/touch /var/www/html/testfile` zadziałają,
 - `sudo -i` oraz `sudo -u www-data /usr/bin/touch /root/hackfile` nie zadziałają, bo nie ma stosownego zapisu w sudoers,
 - secure_path zabezpiecza wyszukiwanie binariów, ścieżka użytkownika w tym przypadku zostanie wyczyszczona i nadpisana przez bezpieczną ścieżkę.

 Sugerowane działania:
 - konfiguracja jest poprawna nie wymaga działań. 
