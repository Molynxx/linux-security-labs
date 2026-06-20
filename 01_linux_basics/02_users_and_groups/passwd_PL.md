# /etc/passwd

## Cel
Celem tego laboratorium było zapoznanie się z zawartością pliku passwd oraz rolą jaką pełni w systemie.
 
## Czym jest /etc/passwd
Plik `/etc/passwd` to plik tekstowy (tzw. flat-file czyli - każdy użytkownik zajmuje jeden osobny wiersz a pola są oddzielone dwukropkami ':'), zawierający podstawowe informacje o wszystkich użytkownikach systemu Linux takie jak:
- nazwa użytkownika, 
- hash hasła lub marker 'x',
- UID (User ID),
- GID (Group ID),
- GECOS (opis użytkownika),
- katalog domowy,
- powłoka logowania (shell).  
W nowoczesnych systemach plik ten nie przechowuje haseł, zamiast hashy, gdy hasło jest ustawione, w pliku występuje marker 'x' a hash hasła jest zapisany w osobnym pliku `/etc/shadow`.  
Plik passwd jest używany przez:
- system logowania,
- procesy identyfikujące użytkownika (UID),
- polecenia takie jak id, ls, ps,
- usługi systemowe.  
Uprawnienia pliku muszą być ustawione tak, żeby był on odczytywalny dla wszystkich (-rw-r--r--) - root może zapisywać, wszyscy inni czytać. Powodem takich uprawnień jest to, że system musi być w stanie:
- przetłumaczyć UID na nazwę użytkownika, 
- wyświetlić właściciela pliku (`ls -l`),  
- umożliwić działanie procesów działających jako różni użytkownicy. 
Gdyby plik nie był world-readable:
- polecenia takie jak `ls`, `who`, `ps` nie mogłyby działać poprawnie, 
- system nie mógłby rozwiązywać UID na nazwy.  
Plik ten jest powiązany z plikami:
- `/etc/shadow` - hasła i polityka haseł,
- `/etc/group` - grupy,
- `/etc/login.defs` - zakresy UID (0 - root, 1-999 użytkownicy systemowi, >=1000 użytkownicy. Zakresy te mogą się różnić w zależności od dystrybucji),  
- PAM - kontrola uwierzytelniania, autoryzacji, sesji, zmiany hasła.  
Należy pamiętać, że plik passwd przechowuje wyłącznie dane kont lokalnych. Jednak nowoczesne systemy są wyposażone w NSS (Name Service Switch). NSS to część systemu wbudowana w libc (bibliotekę standardową systemu). Jej zadaniem jest określenie skąd system będzie pobierał informacje o użytkownikach, grupach, hostach, hasłach, itp. Domyślne informacje mogą pochodzić z:
- plików lokalnych (`/etc/passwd`, `/etc/group`, `/etc/shadow`),
- LDAP (Lightweight Directory Access Protocol) - to centralna baza użytkowników i grup, która może być używana w dużych firmach, żeby wszyscy użytkownicy logowali się do jednego katalogu,
- NIS (Network Information Service) - starszy program centralizacji kont, mniej bezpieczny lecz wciąż spotykany w starych systemach,
- AD (Active Directory) - to microsoftowa centralna baza użytkowników i polityk. Na Linux można używać np. sssd do integracji z AD.  
Mechanizm NSS decyduje, które źródła sprawdza w jakiej kolejności, dzięki konfiguracji w pliku `/etc/nsswitch.conf`. To pozwala uzyskać pełną widoczność logiczną tożsamości w systemie.  
Jak to działa w systemie:  
	Polecenie:     
	`getent passwd student`  
	Kiedy np. polecenie id student pyta system kim jest student, libc używa NSS, żeby sprawdzić:
	- czy student jest w lokalnym `/etc/passwd`,
	- czy student jest w LDAP /AD / NIS (jeśli takie źródła są skonfigurowane).
	To oznacza, że pliki lokalne nie muszą zawierać wszystkich użytkowników, jeśli system korzysta z LDAP lub AD. Polecenie nie przegląda tylko `/etc/passwd`, ale pyta NSS o źródła i zwraca wszystkie konta użytkowników dostępne w systemie, niezależnie od źródła. 

## Struktura wpisów w pliku
Przykład:  
	`student:x:1001:1001:Jan Kowalski:/home/student:/bin/bash`   
Format:   
	`username:password:UID:GID:GECOS:Home_directory:shell`  
Znaczenie pól:
- username - nazwa konta,
- password: 
	- `x` -> hasło znajduje się w /etc/shadow,
	- `*`, `!`, `!!`, `*!` - konto zablokowane.  
- UID - User ID"
	- 0 - root,
	- 1-999 - konta systemowe (zależne od dystrybucji),
	- >= 1000 - konta interaktywne.
- GID - ID głównej grupy użytkownika,
- GECOS - opis użytkownika (imię, telefon, pokój, itp). Pole GECOS może być puste lub zawierać informację wprowadzającą w błąd - atakujący może celowo podać fałszywy opis, by konto wyglądało na legalne,
- Home_directory - np. `/home/student`,
- shell logowania:
	- `/bin/bash`,
	- `/bin/zsh`,
	- `/usr/sbin/nologin`,
	- `/bin/false`.  
	Pierwsze dwa dotyczą powłok charakterystycznych dla użytkowników interaktywnych, pozostałe dwie są charakterystyczne dla kont systemowych.

## Polecenia modyfikujące plik passwd
- `useradd` (tworzenie nowego użytkownika w systemie):
	- tworzy wpis w `/etc/passwd`, `/etc/group` oraz `/etc/shadow` (bez ustawionego hasła),
	- tworzy katalog domowy (jeśli zostanie użyta flaga -m),
	- można od razu przypisać powłokę (`-s /bin/bash`) lub UID/GID.    
	Przykład:    
		`sudo useradd -m -s /bin/bash student`    
	Efekt - w passwd pojawia się linia:  
		`student:x:1001:1001::/home/student:/bin/bash`   
	Zagrożenie: atakujący z dostępem do sudo lub roota może dodać konto z UID 0 -> backdoor.
- `usermod` (modyfikacja istniejącego konta):
	- zmiana powłoki `-s /bin/bash`,
	- zmiana katalogu domowego `-d /home/newdir`,
	- zmiana UID -u,
	- zmiana GID -g,
	- dodanie do grup -aG.  
	Przykład:  
		`sudo usermod -s /bin/bash postgres`    
	Efekt: jeśli np. konto systemowe usługi (UID<1000) dostanie powłokę interaktywną, może to być wektor do backdoora.
- `userdel` (usunięcie użytkownika z systemu):
	- opcja `-r` usuwa także katalog domowy użytkownika.  
	Przykład:  
		`sudo userdel -r student`   
	Efekt: zostanie usunięte konto użytkownika student oraz jego katalog domowy.
	Zagrożenie: atakujący może usuwać konta użytkowników interaktywnych, żeby ukryć swoje działania lub zablokować prawdziwych użytkowników.
- `chsh` (zmiana powłoki logowania użytkownika):  
	Przykład:  
		`sudo chsh -s /bin/bash www-data`   
	Efekt: konto systemowe może nagle mieć powłokę interaktywną.
	Zagrożenie: jeśli konto usługowe dostanie powłokę interaktywną, może zostać użyte przez atakującego do uzyskania dostępu do systemu.
- passwd to polecenie pozwalające na ustawienie lub zmianę hasła użytkownika. Z tym, że hasło nie trafia do passwd lecz do pliku `/etc/shadow`.  
	Przykład:   
		`sudo passwd student`   
	Zagrożenie: atakujący z uprawnieniami sudo może zmienić hasło dowolnego konta -> pełny dostęp do konta, dlatego ważna jest kontrola sudoers. 
Powyższe polecenia wpływają na pliki:
- `useradd` -> passwd + shadow +group,
- `usermod` -> zależnie od opcji,
- `userdel` -> usuwa wpisy z passwd i shadow,
- `chsh` -> tylko passwd,
- `passwd` -> modyfikuje wyłącznie `/etc/shadow`.
Dlatego każda zmiana powinna być monitorowana przez auditd lub inne narzędzia, żeby wykryć nieautoryzowane modyfikacje. 

## Typy użytkowników
- root
	- UID 0,
	- pełne uprawnienia systemowe,
- użytkownicy systemowi 
	- UID <1000
	- często shell /usr/sbin/nologin lub /bin/false.
	Służą do uruchamiania usług (np. www-data, postgres).
- użytkownicy interaktywni
	- UID >=1000,
	- mają zwykle katalog domowy,
	- mają interaktywną powłokę (np. /bin/bash).

## Monitoring / SOC perspective
Celem monitoringu jest wykrywanie nieautoryzowanych zmian w `/etc/passwd` i `/etc/shadow`.
- Co monitorować:
	- tworzenie nowych użytkowników (`useradd`),
	- modyfikacje kont (`usermod`),
	- usuwanie użytkowników (`userdel`),
	- zmiany powłok (`chsh`),
	- zmiany haseł (`passwd`).
- Narzędzia:
	- auditd -> śledzenie syscall open, write dla passwd/shadow,
	- syslog / journald -> logowanie zmian przez sudo,
	- alerty na UID=0 -> wykrycie nieautoryzowanego konta root.

## Wykonane kroki
- Przejrzano plik poleceniem cat `/etc/passwd`,
- Sprawdzono ostatnie logowania użytkowników za pomocą polecenia last

## Obserwacje
- Występują trzy rodzaje użytkowników:
	- root - konto administracyjne z dostępem interaktywnym,
	- użytkownicy systemowi - UID < 1000, powłoka `/usr/sbin/nologin` lub `/bin/false`,
	- zakresy UID mogą się różnić w zależności od dystrybucji i konfiguracji systemu,
	- użytkownicy interaktywni - UID > 1000, powłoka `/bin/bash`, `/bin/zsh` lub `/bin/sh`, katalog domowy w `/home`.
- Niektóre konta systemowe mogą mieć interaktywny shell (np. postgres) - to są wyjątki.

## Wnioski bezpieczeństwa
- Każdy użytkownik systemowy z możliwością interaktywnego logowania wymaga weryfikacji.
- Należy sprawdzić:
	- czy konto zostało zgłoszone i utworzone w sposób zgodny z polityką, 
	- kiedy zostało założone,
	- kiedy i skąd logowało się konto (last, ssh, lokalnie),
	- czy logowania były w nietypowych godzinach.
- Użytkownicy z UID >= 1000 zwykle mają powłokę interaktywną i katalog domowy; każde odstępstwo wymaga uwagi.
- UID 0 oznacza konto root - obecność dodatkowych kont z UID 0 jest bardzo niebezpieczna i wymaga natychmiastowej weryfikacji. 
- Konto systemowe z interaktywnym shellem (`/bin/bash`, `/bin/zsh`) wymaga weryfikacji.
- Plik passwd jest wrażliwy na enumerację (zbieranie informacji o użytkownikach), ponieważ musi być world-readable. Dla atakującego to istotne, bo nie da się zaatakować nieistniejącego konta, a z tego pliku można wyczytać wiele informacji:
	- root, 
	- student,
	- postgres,
	- backup,
	- devops.
	Informacje z tego pliki to duże ułatwienie dla atakującego. 
- Należy monitorować (auditd) ten plik pod kątem wprowadzenia zmian, weryfikować jakie dane zostały wprowadzone, jakie dane zostały zmienione oraz kiedy i kto przeprowadził modyfikację. W przypadku gdy atakujący dostanie się na konto z uprawnieniami root może:
	- dopisać użytkownika z uprawnieniami root,
	- dodać powłokę interaktywną do jakiejś usługi,
	- zmienić UID istniejącego konta użytkownika na 0.
	Wszystkie powyższe czynności prowadzą do pozostawienia backdoora.
- Należy kontrolować zawartość pliku sudoers, w którym, celem zabezpieczenia przed nieautoryzowanymi zmianami m.in. pliku passwd, można ustawić:
	- dostęp do konkretnych poleceń,
	- brak dostępu do edycji plików systemowych, 
	- brak dostępu do vipw (bezpieczny w kwestii integralności systemu edytor pliku passwd, jednak jako root lub sudo plik passwd można zmieniać bez ograniczeń nawet w notatniku. Żeby się przed tym zabezpieczyć, najlepiej wskazać w sudoers jakie polecenia sudo może wykonywać dany użytkownik).
- Warto konfigurować NSS, by za pomocą polecenia getent passwd móc monitorować wszystkich użytkowników, nie tylko lokalnych zapisanych w pliku /etc/passwd,
- warto również sprawdzać integralność plików `/etc/passwd` i `/etc/shadow` (czyli dublujące się UID, nieprawidłowe wpisy, problemy z formatowaniem) za pomocą polecenia: `pwck`.


## Podsumowanie
- Każda nieautoryzowana zmiana -> potencjalny backdoor.
- Monitoring pozwala wykryć nieautoryzowane działania zanim atakujący uzyska trwały dostęp. 
- Połączenie wiedzy o strukturze pliku + audyt daje pełny obraz bezpieczeństwa.

## Analiza wpisów w pliku /etc/passwd (case study)

### Case study 1
```
Fragment passwd:  
root:x:0:0:root:/root:/bin/bash  
admin:x:0:0:System Administrator:/home/admin:/bin/bash  
backup:x:34:34:/var/backups:/usr/sbin/nologin  
```
Kontekst:  
- Server produkcyjny,
- admin jest kontem używanym przez zespół IT,
- System działa od kilku lat.  
   
 Analiza:
 Pierwszy i trzeci wpis jest naturalny. Z kontekstu wynika, że ten system działa w tym ustawieniu od kilku lat. Są jednak tutaj zagrożenia:  
 - obecność dodatkowego konta z UID 0 zwiększa powierzchnię ataku,
 - konto z UID 0 posiada pełne uprawnienia systemowe, niezależnie od nazwy użytkownika, 
 - ponadto konto jest współdzielone, więc trudniej dopasować do odpowiedzialności danej osoby,
 - bezpieczniejszą formą jest stworzenie grupy użytkowników z uprawnieniami sudo na potrzeby zespołu IT,
 - w przypadku braku dokumentacji lub kontroli może stanowić mechanizm trwałego dostępu (persistence).  
  
 Sugerowane działania:  
 - przeprowadzić audyt osób z zespołu IT (czy każda z tych osób w istocie potrzebuje pełnych uprawnień systemowych),
 - weryfikacja polityki dostępu zespołu IT,
 - rozważenie użycia kont sudo zamiast konta admin z UID=0,
 - jeśli jednak postanowiono korzystać z konta admin w takiej formie (niezalecane), należy monitorować logowania, użycie sesji oraz modyfikacje systemowe wykonane z tego konta (np. przez auditd lub SIEM).

 ### Case study 2 
```
 Fragment passwd:  
 www-data:x:33:33:www-data:/var/www:/bin/bash
 deploy:x:1002:1002:Deployment User:/home/deploy:/bin/bash  
```
 Kontekst:
 - Serwer webowy,
 - Aplikacja działa jako www-data,
 - deploy służy do CI/CD  

 Dodatkowe informacje: Konto deploy to konto techniczne. Jest używane automatycznie przez system wdrożeniowy, a nie przez człowieka, do normalnej pracy.
 CI - Continuous Integration   
 Automatycznie:  
 - budowanie aplikacji,
 - testowanie,
 - sprawdzanie jakości kodu,
 - po każdym commicie.  
 CD - Continuous Deployment /Delivery  
 Automatycznie:  
 - wdrażanie aplikacji na serwer,
 - aktualizowanie wersji,
 - restart usług.  
 Konto deploy jest bezpieczne jeśli:  
 - ma ograniczony dostęp tylko do katalogu aplikacji, 
 - nie ma UID =,
 - nie ma sudo,
 - nie ma interaktywnego dostępu dla ludzi,
 - klucze są zarządzane centralnie,
 - jego działania są logowane.   
 Nie jest bezpieczne gdy:  
 - ma sudo ALL,
 - ma UID 0,
 - ma hasło znane kilku osobom,
 - może zmieniać konfigurację systemu.  

 Analiza:  
- Konto systemowe www-data nie powinno posiadać interaktywnej powłoki, jeśli nie ma takiej potrzeby operacyjnej,
- konto deploy wygląda zwyczajnie jednak w pliku passwd nie ma dość informacji by sprawdzić czy na pewno nie jest niebezpieczne.  
Sugerowane działania:  
- zmienić powłokę `/bin/bash` dla usługi www-data na `/usr/sbin/nologin` lub `/bin/false`,
- sprawdzić w `/etc/group` do jakich grup należy konto deploy, gdyż niektóre grupy mogą dawać mu dodatkowe uprawnienia,
- sprawdzić czy ma uprawnienia sudo za pomocą polecenia sudo -l, 
- sprawdzić plik `/etc/sudoers`, `/etc/sudoers.d` pod kątem ograniczeń nałożonych na konto deploy.  
Jeśli konto deploy nie posiada nieuzasadnionych uprawnień oraz jego dostęp jest ograniczony do katalogu aplikacji zgodnie z zasadą najmniejszych uprawnień, ryzyko jest akceptowalne.

### Case study 3
```
Fragment passwd:  
syslog:x:104:110::/home/syslog:/bin/bash 
 ```
Kontekst:  
- Konto wcześniej miało `/usr/sbin/nologin`,
- Zmiana została zauważona po 2 tygodniach,
- Brak innych alertów.  

Analiza:  
- zmiana powłoki usługi odpowiedzialnej za logowanie zdarzeń z `/usr/sbin/nologin` na `/bin/bash` zawsze budzi niepokój i wymaga natychmiastowego sprawdzenia,
- konta systemowe nie powinny mieć dostępu do interaktywnej powłoki,
- konta systemowe nie powinny mieć katalogu domowego w postaci `/home/nazwa_usługi`, mogą mieć katalog techniczny ale on zwykle znajduje się w katalogu `/var` lub `/srv`,
Wpis ten wygląda jak interaktywne konto, jednak to jest konto systemowe, któremu dodano katalog domowy i interaktywną powłokę. Może to oznaczać potencjalne naruszenie bezpieczeństwa lub uzyskanie nieautoryzowanego dostępu.   

Sugerowane działania:  
- Shadow: sprawdzić `/etc/shadow`, czy konto jest zablokowane i czy istnieje hasło,
- Autoryzacja: zweryfikować, czy konto zostało formalnie autoryzowane,
- Historia zmian: sprawdzić historię zmian w `/etc/passwd`, szczególnie powłoki i nowe konta. 
- Grupy: sprawdzić przynależności do grup użytkownika,
- Sudo: sprawdzić uprawnienia sudo (`sudo -l -U syslog`) oraz wpisy w `/etc/sudoers` i `/etc/sudoers.d`,
- Audyt: jeśli konto nie było udokumentowane ani autoryzowane, jeśli posiada uprawnienia sudo należy niezwłocznie przywrócić prawidłowy katalog techniczny oraz zmienić powłokę na `/usr/sbin/nologin`,
- Kontrola zmian: przejrzeć pliki `/etc/passwd`, `/etc/sudoers`, `/etc/shadow` pod kątem podobnych zmian (zmiana powłoki na innych kontach, dodanie użytkowników),
- PAM: sprawdzić zmiany w plikach PAM, które definiują polityki dostępu,
- Reset haseł: jeśli istnieje podejrzenie wycieku lub nieautoryzowanych zmian, należy zresetować hasła wszystkich kont krytycznych, aby ograniczyć ryzyko utraty kontroli nad systemem.   

Monitoring:  
- Skonfigurować monitorowanie krytycznych plików (`/etc/passwd`, `/etc/shadow`, `/etc/group`, `/etc/sudoers`) przy uznaniu auditd lub SIEM, tak aby każda zmiana generowała natychmiastowy alert,
- zapewnić, że alerty trafiają do odpowiedzialnego zespołu SOC/Blue Team,
- ustawić odpowiednie reguły, które zwracają uwagę na zmiany powłoki, dodawanie kont z UID 0 lub nowe wpisy sudo.   
Taki nietypowy dostęp może świadczyć o potencjalnym persistence lub backdoor. 


