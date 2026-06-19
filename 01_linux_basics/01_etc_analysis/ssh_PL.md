# /etc/ssh

## Cel laboratorium 
Celem tego laboratorium było zaznajomienie się z mechanizmami zdalnego logowania, konfiguracji SSH i niebezpieczeństw z tym związanych. 

## Czym jest /etc/ssh
Jest to katalog przechowujący pliki konfiguracyjne dla usługi SSH (Secure Shell), służącej do zdalnego logowania do systemu. Główny plik konfiguracyjny serwera SSH znajduje się w ścieżce: /etc/ssh/sshd_config. W pliku można zdefiniować m.in.:
- sposób łączenia się z systemami zdalnymi,
- port i adresy nasłuchu,
- metody uwierzytelniania,
- uprawnienia użytkowników i grup.
Plik sshd_config jest przetwarzany od góry do dołu zgodnie z kolejnością wpisów. Linie rozpoczynające się znakiem # są komentarzami. Zawierają one informacje pomocnicze oraz przykładowe opcje, które nie są aktywne, dopóki nie zostaną odkomentowane lub nadpisane. Odkomentowanie lub dodanie opcji powoduje zastąpienie wartości domyślnej. 
Dobrą praktyką administracyjną jest dodawanie własnych ustawień na końcu pliku, zamiast edytowania istniejących wpisów - ułatwia to audyt i analizę zmian.
Warto podkreślić, że plik sshd_config nie pokazuje domyślnych ani aktualnie obowiązujących ustawień SSH. Aby sprawdzić rzeczywistą (efektywną) konfigurację, należy użyć polecenia: sudo sshd -T. Polecenie to wyświetla efektywną konfigurację SSH, czyli wartości domyślne połączone z tym, co zostało nadpisane w plikach konfiguracyjnych. 

## HostKey - fundament zaufania SSH 
Wprawdzie dotyczy zaufanych hostów, które klient zapisuje w katalogu `~/.ssh/known_hosts`, lecz to ma ścisły związek z połączeniem SSH, dlatego warto o tym wspomnieć przy analizie tego katalogu. Jak działa HostKey:
- często dla zwiększenia bezpieczeństwa, pracownicy są informowani przez admina, jaki jest fingerprint hosta. Fingerprint to skrót kryptograficzny (hash) klucza publicznego hosta. Jest on wyświetlany użytkownikowi podczas pierwszego połączenia, aby można było zweryfikować tożsamość serwera. 
- podczas pierwszego logowania przez SSH, host wysyła klientowi swój klucz publiczny,
- klient przelicza klucz publiczny hosta na hash, który pokazuje użytkownikowi, 
- jeśli się zgadza, to po wpisaniu 'yes', klucz publiczny hosta zostaje zapisany w katalogu `~/.ssh/known_hosts` w systemie użytkownika,
- podczas kolejnych logowań klient sprawdza zgodność podawanego każdorazowo przy logowaniu klucza z tym zapisanym w katalogu known_hosts,
- jeśli ten klucz się nie zgadza wyświetla się ostrzeżenie o niezgodności, co może oznaczać:
	- reinstalację serwera,
	- rotację klucza, 
	- atak MITM,
	- kompromitację.
Takie rozwiązanie umożliwiające początkową weryfikację hosta mocno ogranicza zagrożenie MITM, jeśli klucz się zmieni, klient SSH wyświetli ostrzeżenie, ponieważ zapisany fingerprint nie będzie się zgadzał. Może to oznaczać zmianę klucza lub potencjalny atak MITM. 

## Autoryzowane klucze użytkownika
Będąc przy temacie HostKey warto też wspomnieć o kluczach użytkownika. Klucze publiczne, które mogą logować się na konto użytkownika, przechowywane są w pliku systemu serwera:  
	`~/.ssh/authorized_keys`   
Jeśli atakujący uzyska dostęp do konta z uprawnieniami zapisu do tego pliku, może dodać własny klucz publiczny i uzyskać trwały dostęp do serwera bez znajomości hasła. Dlatego w analizie incydentów bezpieczeństwa jednym z pierwszych kroków jest sprawdzenie zawartości pliku `authorized_keys`. Można zmienić lokalizacje przechowania zaufanych kluczy w pliku `sshd_config`.

## Match Blocks - warunkowe reguły SSH
Match blocks występują w pliku sshd, tam gdzie wszystkie pozostałe ustawienia SSH. Match block to reguły warunkowe. Plik sshd_config jest przetwarzany z góry do dołu, match block mówi: 'jeśli te warunki zostaną spełnione, nadpisz wcześniejsze ustawienia w pliku'. Przykład  - fragment pliku: 
```
	PasswordAuthentication yes  
	PermitRootLogin no  
	Match User admin  
		PasswordAuthentication no  
```
to oznacza, że:  
- hasła są dozwolone dla wszystkich poza rootem,
- dla usera admin PasswordAuthentication 'yes' zostaje nadpisane przez 'no', to blokuje logowanie hasłem i wymusza logowanie kluczem, co zwiększa bezpieczeństwo.  
Jest to ważne, ponieważ jeśli zostanie popełniony błąd np. wyłączyć uwierzytelnianie dla roota, jeśli warunek jest źle zdefiniowany.  Podczas analizy logów trzeba sprawdzić, który blok zadziałał dla danego połączenia. 

## Logowanie i LogLevel
Każda próba logowania jest zapisywana w logach na serwerze (/var/log/auth.log, systemd). W pliku config w opcji LogLevel można ustawić poziom szczegółów wyświetlanych w logach. Jest kilka poziomów, zbyt niski nie pokaże żadnego logowania, zbyt wysoki może sprawić, że logi będą przeładowane detalami, co może utrudnić audyt.   
Poziomy szczegółowości:
- QUIET - nie pokazuje prawie nic, więc z punktu widzenia SOC jest prawie bezużyteczny, 
- FATAL/ERROR - wyłącznie błędy krytyczne, nie widać prób logowania,
- INFO - normalne informacje, można zobaczyć kto się logował i z jakiego IP,
- VERBOSE - bardzo szczegółowo, pokazuje np. odcisk klucza publicznego, metodę logowania,
- DEBUG - wszystko, nawet debug dla dev, bardzo niepraktyczny dla SOC ze względy na dużą ilość szumu informacyjnego.

## Wnioski bezpieczeństwa
Z perspektywy bezpieczeństwa SOC, w środowisku produkcyjnym powyższe ustawienia wymagały by dodatkowej konfiguracji w celu ograniczenia ryzyka.
- Domyślny port 22 jest powszechnie znany i często skanowany. Zmiana portu na niestandardowy (niekolidujący z innymi usługami) może ograniczyć automatyczne ataki typu brute force. Numery portów:
	- 0 - 1023 - well-known, lepiej nie używać, 
	- 1024 - 49151 - registered, możliwie konflikty,
	- 49152 - 65535 - dynamic, najlepszy zakres przy wyborze numeru portu. 
- Nasłuchiwanie na wszystkich adresach IPv6 ([::]) nie zawsze jest konieczne. Jeśli IPv6 nie jest używane, warto rozważyć ustawienie AddressFamily na inet, aby ograniczyć nasłuch wyłącznie do IPv4. 
- Nasłuchiwanie na 0.0.0.0 jest poprawne funkcjonalnie, jednak z punktu widzenia bezpieczeństwa można ograniczyć powierzchnię ataku poprzez przypisanie SSH do konkretnego interfejsu lub adresu IP,
- Ustawienie PermitRootLogin prohibit-password jest dobrą praktyką uniemożliwia logowanie roota za pomocą hasła, co znacząco ogranicza ryzyko brute force,
- PasswordAuthentication yes stanowi potencjalne zagrożenie bezpieczeństwa, bezpieczniejszym rozwiązaniem jest wyłączenie logowania hasłem i korzystanie wyłącznie z kluczy SSH.
- PubKeyAuthentication yes jest ustawieniem bezpiecznym i zalecanym, 
- PermitEmptyPasswords no jest najlepszym możliwym ustawieniem - logowanie bez hasła znacząco zwiększa powierzchnię ataku, 
- Brak konfiguracji AllowUsers / DenyUsers oraz AllowGroups / DenyGroups oznacza brak ograniczeń dostępu. Bezpieczniejszym podejściem jest jawne wskazanie użytkowników lub grup, które mogą korzystać z SSH, pamiętając, że reguły deny* mają pierwszeństwo,
- MaxAuthTries ustawione na 6 może ułatwić brute force, zalecaną wartością jest 3, z uwzględnieniem możliwości pomyłki użytkownika,
- UsePam yes oznacza, że SSH korzysta z mechanizmów PAM, dlatego ograniczenia logowania mogą wynikać z polityk PAM, a nie tylko z ustawień SSH.  
	Co się dzieje przy logowaniu:
	- klient wpisuje login / hasło lub używa klucza,
	- sshd sprawdza, czy metoda jest dozwolona (PasswordAuthentication yes/no),
	- sshd wywołuje PAM (pam_start() w kodzie),
	- PAM odpytuje każdy moduł w kolejności w pliku /etc/pam.d/sshd),
	- jeśli każdy moduł 'akceptuje' - logowanie jest udane,
	- jeśli którykolwiek blokuje - logowanie jest odrzucone. 

## Najczęstsze misconfigi SSH
- PermitRootLogin yes - zawsze źle na publicznym serwerze, 
- PasswordAuthentication yes - umożliwia brute force,
- Brak AllowUsers - każdy może próbować loginów, 
- Brak limitów / fail2ban - brak automatycznej obrony, 
- stare ciphers/MAC - podatne na MITM,
- PermitEmptyPasswords yes - bardzo niebezpieczne ustawienie pozwalające na pominięcia hasła.

## Kontekst SOC
Zmiany w konfiguracji SSH, uruchamianie lub zatrzymywanie usługi SSH oraz wielokrotnie nieudane próby logowania są zdarzeniami istotnymi z punktu widzenie SOC. Powinny one być monitorowane i korelowane z logami systemowymi (journalctl, auth.log) oraz alertami dotyczącymi brute force, nieautoryzowanego dostępu lub nieautoryzowanych zmian konfiguracyjnych.
- Należy monitorować auth.log i journalctl pod kątem prób brute force oraz nieautoryzowanych zmian w konfiguracji SSH,
- Warto regularnie sprawdzać efektywną konfigurację SSH (sudo sshd -T) po każdej zmianie plików konfiguracyjnych,

## Analiza konfiguracji SSH (Case Study)

### Case Study 1
```
Przykład konfiguracji SSH:  
	Port 22  
	ListenAddress 0.0.0.0  
	PermitRootLogin yes   
	PasswordAuthentication yes
	PubKeyAuthentication no  
	AllowUsers alice bob  
	MaxAuthTries 10  
	ClientAliveInterval 300  
	ClientAliveCountMax 3  
```
Analiza:  
- ustawiony port jest dobrze znany atakującym, jest często skanowany, 
- nasłuch ustawiony jest na wszystkich interfejsach IPv4 (0.0.0.0). Jeśli serwer posiada wiele interfejsów sieciowych, ograniczenie nasłuchu do konkretnego zakresu IP może zmniejszyć powierzchnię ataku,
- PermitRootLogin yes to zawsze niebezpieczny wybór, zwłaszcza na serwerach publicznych, 
- PasswordAuthentication yes, opcja pozwala użytkownikom na logowanie za pomocą hasła. W środowiskach produkcyjnych nie jest to zalecanie, ponieważ zwiększa ryzyko brute force oraz wycieku lub złamania hasła,
- PubKeyAuthentication no, wyłączenie opcji logowania kluczem jest niebezpieczne, ponieważ wtedy pozostaje tylko opcja logowania hasłem, co nie należy do najbezpieczniejszych sposobów logowania, 
- AllowUsers, to jest dobra praktyka, by ograniczać dostęp tylko do określonej grupy użytkowników, 
- MaxAuthTries, 10 to zdecydowanie zbyt wysoka wartość, zwiększa ryzyko brute force,
- ClientAliveInterval 300, to poprawne ustawienie, oznacza, że serwer co 5 minut sprawdza czy klient jest nadal aktywny,
- ClientAliveCountMax 3, to rozsądne ustawienie, ponieważ jeśli klient nie odpowie 3 razy, to biorąc pod uwagę opcję zapisaną powyżej, serwer przerwie połączenie po 15 minutach.    

Sugerowane działania:  
- należy zmienić port na mniej oczywisty, warto używać zakresu 49152 - 65535 żeby unikać kolizji i zmniejszyć ryzyko skanowania, 
- opcjonalnie, celem zwiększenia bezpieczeństwa ograniczyć nasłuch do określonego IP,
- należy zmienić PermitRootLogin na 'no', zwłaszcza jeśli serwer jest publiczny,
- należy zmienić autoryzację hasłem na 'no' oraz autoryzację kluczem na 'yes'. Klucze są znacznie bezpieczniejsze i bardziej odporne na brute force,
- należy zmienić dozwoloną ilość prób logowania na 3, to dobry kompromis pomiędzy ochroną przed brute force a możliwości pomyłki przy wpisywaniu hasła przez użytkownika. 

### Case Study 2
```
Konfiguracja:  
	Port 2222  
	PermitRootLogin prohibit-password  
	PasswordAuthentication no  
	PubKeyAuthentication yes  
	AllowUsers admin  
	MaxAuthTries 6  
```
Analiza:  
- port 2222 zmniejsza liczbę automatycznych skanów i prób logowania do domyślnym porcie 22, jednak nie stanowi rzeczywistego mechanizmu bezpieczeństwa,
- AllowUsers - ustawienie dostępu tylko dla jednego konta:
	- zmniejsza powierzchnię ataku,
	- zmniejsza liczbę potencjalnych punktów kompromitacji,
	- zmniejsza szum informacyjny w logach,
	jednak są tutaj też negatywne skutki, ponieważ gdy: 
	- klucz admina zostanie skradziony, 
	- laptop admina zostanie zhakowany,
	- klucz admina wycieknie z GitHub,
	- ktoś przejmie konto admina,
	to atakujący ma pełny dostęp do serwera, bo nie ma żadnego innego konta kontrolnego, natomiast jeśli:
	- klucz admina zostanie uszkodzony, niepoprawnie zostanie zdefiniowane AllowUsers, zmieni się konfiguracja SSH, to serwer jest zablokowany administracyjnie,
	- tego jednego konta używa kilku adminów utrudniony jest audyt, nie wiadomo, który admin jakie procesy uruchomił. Dlatego w tym przypadku zawsze warto ustawić dostęp dla grupy adminów, dzięki czemu można uniknąć powyższych konsekwencji. 
- PermitRootLogin  jest w tym przypadku zbędne, ponieważ AllowUsers i tak wyklucza roota, jednak warto je zostawić jako dodatkową warstwę zabezpieczeń. 
- MaxAuthTries - ustawiona wartość jest zbyt duża, może zwiększyć ryzyko brute force.  

Sugerowane działania:  
- ustawić więcej kont, celem zabezpieczenia dostępu do serwera w razie przejęcia konta, uszkadzania konfiguracji SSH lub klucza oraz dla ułatwienia audytu,
- opcję PermitRootLogin można pozostawić jako dodatkową warstwę zabezpieczeń lub usunąć, ponieważ AllowUsers i tak wyklucza roota,
- zmienić MaxAuthTries na wartość 3, co zwiększy bezpieczeństwo przed brute force, ale jednak uwzględnia pomyłkę w haśle przez użytkownika. Dodatkowo liczba prób logowania powinna być monitorowana w logach systemowych (np/ auth.log), ponieważ duża liczba nieudanych prób może wskazywać na atak brute force.

### Case Study 3
```
Konfiguracja:  
	Zakładamy środowisko produkcyjne z SSH:  
	Port 2222  
	PermitRootLogin no  
	PasswordAuthentication no 
	PubKeyAuthentication yes 
	AllowUsers admin  
	MaxAuthTries 3  
```
Celem jest monitorowanie logów auth.log lub journalctl w poszukiwaniu nieprawidłowych zachowań, brute force i potencjalnych kompromitacji.  
```
Log 1   
Mar 6 09:15:23 kali sshd[2345]: Accepted publickey for admin from 192.168.1.10 port 54321 ssh2: RSA SHA256:abcd1234....  

Log 2  
Mar 6 03:42:11 kali sshd[2350]: Failed password for admin from 185.203.56.77 port 42213 ssh2  
Mar 6 03:42:15 kali sshd[2350]: Failed password for admin from 185.203.56.77 port 42214 ssh2  
Mar 6 03:42:19 kali sshd[2350]: Failed password for admin from 185.203.56.77 port 42215 ssh2  
Mar 6 03:42:23 kali sshd[2350]: Failed password for admin from 185.203.56.77 port 42216 ssh2  
 
Log 3  
Mar 6 22:17:02 kali sshd[2367]: Accepted publickey for admin from 198.51.100.45 port 59922 ssh2: RSA SHA256:abcd1234....    
```
Analiza:  
- konfiguracja SSH jest ustawiona poprawnie, z wyjątkiem AllowUsers - ma tylko jednego użytkownika, co zwiększa ryzyko administracyjnego zablokowania serwera lub utraty kontroli nad serwerem,
- log 1 - logowanie z systemu kali za pomocą klucza dla użytkownika admin, odbywa się z lokalnego adresu IP, port 54321. Tutaj nic nie wskazuje na to, że dzieje się coś złego. Wygląda to jak zwyczajnie logowanie admina w godzinach pracy z sieci lokalnej,
- log 2 - wiele zakończonych niepowodzeniem prób logowania do konta admin za pomocą hasła co było zabronione w konfiguracji. Logowania odbywają się późno w nocy co 4 minuty z zewnętrznego adresu IP. To bardzo mocno wskazuje na próbę ataku brute force. Admin na pewno wie, że możliwość zalogowania zdalnie odbywa się wyłącznie za pomocą klucza, więc dlaczego miałby próbować logowania za pomocą hasła,   
- log 3 - logowanie po 22 w nocy, odbywające się poza standardowymi godzinami pracy,  powinno budzić podejrzenia, zwłaszcza jeśli to nie są godziny pracy. Teoretycznie to mógł być admin pragnący przeprowadzić jakieś prace na serwerze poza godzinami pracy, to jeszcze nie jest powód do paniki, jednak przypadek wymaga dokładnego sprawdzenia.   

Sugerowane działania:  
- dodać jedno lub dwa konta do AllowUsers,
- sprawdzić zabezpieczenia konfiguracji, upewnić się, że MaxAuthTries ma niską wartość, że hasło nie jest dozwolone podczas logowania, czy logowanie root jest dozwolone oraz jaka jest grupa AllowUsers,
- sprawdzić dokładnie dane z logu 3 - czy IP jest zaufanym adresem, upewnić się czy klucz nie wyciekł, sprawdzić w logach jakie działania przeprowadził admin po zalogowaniu i jak długo był online, skonsultować logi z adminem, celem potwierdzenia, że to on się logował. Ponadto należy sprawdzić fingerprint klucza, czy zgadza się z faktycznym. 