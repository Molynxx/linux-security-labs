# /etc/passwd

## Cel
Celem tego laboratorium było zapoznanie się z zawartością pliku passwd oraz rolą jaką pełni w systemie.
 
### Czym jest /etc/passwd
Plik `/etc/passwd` to plik tekstowy (tzw. flat file), zawierający informacje o wszystkich użytkownikach systemu Linux. W nowoczesnych systemach plik ten nie przechowuje haseł, a zamiast hashy haseł występuje tutaj marker `x` (jeśli hasło zostało ustawione dla konta). Hash hasła jest zapisywany w pliku `/etc/shadow`.  Plik passwd jest używany przez: 
- system logowania, 
- procesy identyfikujące użytkownika (UID),
- polecenia takie jak `id`, `ls`, `ps`, 
- usługi systemowe.
Uprawnienia pliku muszą być ustawione tak, żeby był on odczytywalny dla wszystkich (-rw-r--r--), czyli root może zapisywać, wszyscy inni czytać. Powodem takich uprawnień jest to, że system musi być w stanie:
- przetłumaczyć UID na nazwę użytkownika, 
- wyświetlić właściciela pliku (ls -l),
- umożliwić działanie procesów działających jako różni użytkownicy.
Plik ten jest powiązany z plikami:
- `/etc/shadow` - historia i polityka haseł,
- `/etc/group` - grupy,
- `/etc/login.defs`,
- PAM.
Należy pamiętać, że plik passwd przechowuje wyłącznie dane kont lokalnych. Do bezpiecznej edycji pliku `/etc/passwd` używa się polecenia `vipw`, które blokuje plik przed jednoczesną edycją i sprawdza poprawność zmian. 

## Struktura wpisów w pliku
Przykład:
	`student:x:1001:1001:Jan Kowalski:/home/student:/bin/bash`
Format:
	`username:password:UID:GID:GECOS:Home_directory:shell

## Polecenia modyfikujące plik passwd
- `useradd` - tworzenie nowego użytkownika w systemie,
- `usermod` - modyfikacja istniejącego konta,
- `userdel` - usunięcie użytkownika z systemu,
- `chsh` - zmiana powłoki logowania użytkownika, 
- `passwd` - polecenie pozwalające na ustawienie hasła lub zmianę hasła użytkownika, z tym że hasło nie trafia do passwd lecz do pliku `/etc/shadow`.

## Typy użytkowników
- root 
	- UID 0,
	- Pełne uprawnienia systemowe, 
- użytkownicy systemowi
	- UID <1000
	- często shell `/usr/sbin/nologin` lub `/bin/false`
	Służą do uruchamiania usług (np. www-data, postgres).
- użytkownicy interaktywni
	- UID >=1000
	- mają zwykle katalog domowy
	- mają interaktywną powłokę (np `/bin/bash`).

## Wykonane kroki
- przejrzano plik poleceniem cat /etc/passwd,
sprawdzono ostatnie logowania użytkowników za pomocą polecenia `last`.

## Obserwacje 
- występują trzy rodzaje użytkowników:
	- root - konto administracyjne z dostępem interaktywnym,
	- użytkownicy systemowi UID < 1000, powłoka `/usr/sbin/nologin` lub `/bin/false`, zakresy UID mogą się różnić w zależności od dystrybucji i konfiguracji systemu,
	- użytkownicy interaktywni - UID >= 1000, powłoka `/bin/bash`, `/bin/zsh` lub /bin/sh, katalog domowy w `/home`,
- niektóre konta systemowe mogą mieć interaktywny shell (np. postgres) - to są wyjątki. 

## Wnioski bezpieczeństwa
- każde konto systemowe posiadające interaktywną powłokę wymaga weryfikacji, należy sprawdzić:
	- czy konto zostało zgłoszone i utworzone w sposób zgodny z polityką,
	- kiedy zostało założone,
	- kiedy i skąd logowało się konto (last, ssh, lokalnie),
	- czy logowania były w nietypowych godzinach.
- użytkownicy z UID >= 1000 zwykle mają powłokę interaktywną i katalog domowy, każde odstępstwo wymaga uwagi,
- UID 0 oznacza konto root - obecność dodatkowych kont z UID 0 jest bardzo niebezpieczne i wymaga natychmiastowej weryfikacji,
- plik passwd jest wrażliwy na enumerację (zbieranie informacji o użytkownikach), ponieważ musi być world-readable. Dla atakującego to istotne, bo nie da się zaatakować nieistniejącego konta, a z tego pliku można wyczytać wiele informacji. 
- należy monitorować (auditd) ten plik pod kątem wprowadzania zmian, zweryfikować jakie dane zostały wprowadzone, jakie zostały zmienione oraz kiedy i kto przeprowadził modyfikację. W przypadku gdy atakujący dostanie się na konto z uprawnieniami root może:
	- dopisać użytkownika z uprawnieniami root,
	- dodać powłokę interaktywną do jakiejś usługi,
	- zmienić UID istniejącego konta użytkownika na 0.
	Wszystkie powyższe czynności prowadzą do pozostawienia backdoora.
- należy kontrolować zawartość pliku sudoers , w którym celem zabezpieczania przed nieautoryzowanymi zmianami można ustawić:
	- dostęp do konkretnych poleceń, 
	- brak dostępu do edycji plików systemowych, 
	- brak dostępu do vipw (bezpieczny w kwestii integralności systemu edytora pliku passwd,  jednak jako root lub sudo plik passwd można zmieniać bez ograniczeń nawet w notatniku. żeby się przed tym zabezpieczyć, najlepiej wskazać w sudoers jakie polecenia sudo może wykonywać dany użytkownik).
- warto konfigurować NSS, aby za pomocą polecenia getent passwd móc monitorować wszystkich użytkowników, nie tylko lokalnych zapisanych w pliku `/etc/passwd`. 

## Analiza wpisów w pliku `/etc/passwd` (Case Study)

### Przypadek 1  

Fragment `/etc/passwd`:  
	root:x:0:0:root:/root:/bin/bash  
	admin:x:0:0:System Administrator:/home/admin:/bin/bash  
	backup:x:34:34:/var/backups:/usr/sbin/nologin  

Kontekst:  
	Serwer produkcyjny  
	admin jest kontem używanym przez zespół IT  
	system działa od kilku lat  

Analiza:   
Pierwszy i trzeci wpis jest neutralny, z kontekstu wynika, że system ten działa w tym ustawieniu od kilku lat. Zidentyfikowane zagrożenia:
- obecność dodatkowego konta z UID 0 zwiększa powierzchnię ataku,
- konto z UID 0 posiada pełne uprawnienia systemowe, niezależnie od nazwy użytkownika,
- ponadto konto jest współdzielone, więc trudniej dopasować do odpowiedzialności danej osoby, 
- bezpieczniejszą formą jest stworzenie grupy użytkowników z uprawnieniami sudo na potrzeby zespołu IT,
- w przypadku braku dokumentacji lub kontroli może stanowić mechanizm trwałego dostępu (persistence).  

Sugerowane działania:  
- przeprowadzić audyt osób z zespołu IT (czy każda z tych osób w istocie potrzebuje pełnych uprawnień systemowych),
- weryfikacja polityki dostępu zespołu IT,
- rozważenie użycia kont sudo zamiast konta admin z UID 0,
- jeśli jednak postanowiono korzystać z konta admin w takiej formie (niezalecane), należy monitorować logowania, użycie sesji oraz modyfikację systemowe wykonane z tego konta (np. przez auditd lub SIEM)

### Przypadek 2 

Fragment passwd:  
	www-data:x:33:33:www-data:/var/www:/bin/bash  
	deploy:x:1002:1002:Deployment User:/home/deploy:/bin/bash  

Kontekst:   
	serwer webowy   
	aplikacja działa jako www-data  
	deploy służy do CI/CD   

Dodatkowe informacje:   
Konto deploy to konto techniczne. Jest używane automatycznie przez system wdrożeniowy, a nie przez człowieka, do normalnej pracy.   

Analiza:  
- konto systemowe www-data nie powinno posiadać interaktywnej powłoki, jeśli nie ma takiej potrzeby operacyjnej,
- konto deploy wygląda zwyczajnie, jednak w pliku passwd nie ma dość informacji by sprawdzić czy na pewno nie jest niebezpieczne,   

Sugerowane działania:  
- zmienić powłokę `/bin/bash` dla usługi www-data na `/usr/sbin/nologin` lub `/bin/false`,
- sprawdzić w `/etc/group` do jakich grup należy konto deploy, gdyż niektóre grupy mogą dawać mu dodatkowe uprawnienia, 
- sprawdzić czy konto deploy ma uprawnienia sudo za pomocą polecenia `sudo -l`, `sudo visudo -c`,
- sprawdzić plik `/etc/sudoers`, `/etc/sudoers.d/` pod kątem ograniczeń nałożonych  na konto deploy.
Jeśli konto deploy nie posiada nieuzasadnionych uprawnień oraz jego dostęp jest ograniczony do katalogu aplikacji zgodnie z zasadą najmniejszych uprawnień, ryzyko jest akceptowalne. 

### Przypadek 3 

Fragment passwd:  
	syslog:x:104:110::/home/syslog:/bin/bash   

Kontekst:  
	 konto wcześniej miało `/usr/sbin/nologin`  
	 zmiana została zauważona po 2 tygodniach  
	 brak innych alertów  

Analiza:  
- zmiana powłoki usługi odpowiedzialnej za logowanie zdarzeń z `/usr/sbin/nologin` na `/bin/bash` zawsze budzi niepokój i wymaga natychmiastowego sprawdzenia,
- konta systemowe nie powinny mieć dostępu do interaktywnej powłoki ani katalogu domowego w postaci `/home/nazwa_usługi`, mogą mieć katalog techniczny ale on zwykle znajduje się w katalogu `/var/` lub `/srv`,
Wpis ten sugeruje konto interaktywne, jednak to jest konto systemowe, któremu dodano katalog domowy i interaktywną powłokę. Może to oznaczać potencjalne naruszenie bezpieczeństwa lub uzyskanie nieautoryzowanego dostępu.   

Sugerowane działania:  
- shadow: sprawdzić `/etc/shadow`, czy konto jest zablokowane i czy istnieje hasło,
- autoryzacja: zweryfikować, czy konto zostało formalnie autoryzowane, 
- historia zmian: sprawdzić historię zmian w `/etc/passwd `, szczególnie powłoki i nowe konta,
- grupy: sprawdzić przynależności do grup użytkownika, 
- sudo: sprawdzić uprawnienia sudo `sudo -l -U syslog` oraz wpisy w `/etc/sudoers` i `/etc/sudoers.d`,
- audyt: jeśli konto nie było udokumentowane ani autoryzowane, jeśli posiada uprawnienia sudo należy niezwłocznie przywrócić prawidłowy katalog techniczny oraz zmienić powłokę na `/usr/sbin/nologin`,
- kontrola zmian: przejrzeć pliki `/etc/passwd`, `/etc/sudoers`, `/etc/shadow` pod kątem podobnych zmian (zmiana powłoki na innych kontach, dodanie użytkowników),
- PAM: sprawdzić zmiany w plikach PAM, które definiują polityki dostępu,
- reset haseł: jeśli istnieje podejrzenie wycieku lub nieautoryzowanych zmian, należy zresetować hasła wszystkich kont krytycznych, aby ograniczyć ryzyko utraty kontroli nad systemem.   
Monitoring:
- skonfigurować monitorowanie krytycznych plików (`/etc/passwd`, `/etc/sudoers`, `/etc/shadow`, ` /etc/group`) przy uznaniu auditd lub SIEM, tak aby każda zmiana generowała natychmiastowy alert,
- zapewnić, że alerty trafiają do odpowiedzialnego zespołu SOC/Blue Team,
- ustawić odpowiednie reguły, które zwracają uwagę na zmiany powłoki, dodawanie kont z UID 0 lub nowe wpisy sudo.
Taki nietypowy dostęp może świadczyć o potencjalnym persistence lub backdoor.