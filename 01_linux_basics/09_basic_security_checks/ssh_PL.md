# /etc/ssh

## Cel laboratorium 
Celem tego laboratorium było zaznajomienie się z mechanizmami zdalnego logowania, konfiguracji SSH i niebezpieczeństw z tym związanych.

## Czym jest /etc/ssh
Jest to katalog przechowujący pliki konfiguracyjne dla usługi SSH (Secure Shell), służącej do zdalnego logowania do systemu. Główny plik konfiguracyjny serwera SSH znajduje się w ścieżce: `/etc/ssh/sshd_config` i zawiera m.in. metody uwierzytelniania, dostęp użytkowników, parametry połączenia.
Aby sprawdzić rzeczywistą (efektywną) konfigurację, należy użyć polecenia: sudo sshd -T. Plik może zawierać Match Block (warunkowe reguły) - reguły te nadpisują wcześniejsze ustawienia po spełnieniu warunków. Przykład:
	PasswordAuthentication yes
		Match User admin
			PasswordAuthentication no
Opcja LogLevel określa szczegółowość logów (INFO, VERBOSE, DEBUG).

## Wykonane kroki
- analiza konfiguracji SSH (sshd_config, sshd -T),
- weryfikacja statusu usługi (systemctl),
- sprawdzenie nasłuchu (ss -tulpn).

## Obserwacje
Obserwacje dotyczą domyślnej konfiguracji SSH w Kali Linux.
- W pliku `/etc/ssh/sshd_config` nie zawiera aktywnych niestandardowych ustawień - serwer działa na konfiguracji domyślnej OpenSSH,
- Usługa SSH była domyślnie wyłączona w systemie Kali Linux,
- Po sprawdzeniu efektywnej konfiguracji poleceniem sudo sshd -T stwierdzono, że:
	- Port ustawiony jest na 22, 
	- ListenAddress ustawiony na 0.0.0.0 oraz [::], co oznacza nasłuch na wszystkich adresach IPv4 i IPv6,
	- PermitRootLogin ustawione jest na prohibit-password,
	- PasswordAuthentication ustawiony jest na yes,
	- PubKeyAuthentication ustawiony jest na yes,
	- PermitEmptyPasswords ustawiony jest na no, 
	- AllowUsers oraz DenyUsers nie są skonfigurowane,
	- AllowGroups oraz DenyGroups nie są skonfigurowane, 
	- MaxAuthTries ustawione jest na 6,
	- UsePAM ustawione jest na yes,
- Pomimo ustawienia MaxAuthTries na wartość 6, po trzech nieudanych próbach logowania sesja SSH została przerwana, może to wynikać z mechanizmów PAM (np. faillock) - wymaga potwierdzenia w logach. 

## Wnioski bezpieczeństwa
Z perspektywy SOC w środowisku produkcyjnym powyższe ustawienia wymagałyby dodatkowej konfiguracji w celu ograniczenia ryzyka.
- domyślny port 22 jest powszechnie znany i często skanowany. Zalecana zmiana portu na niestandardowy,co ograniczy skany,
- nasłuchiwanie na wszystkich adresach IPv6 ([::}) jeśli IPv6 nie jest używane, ograniczyć nasłuch wyłącznie do IPv4,  
- nasłuch na 0.0.0.0 zwiększa powierzchnię ataku - zalecane ograniczenie do konkretnego adresu IP/interfejsu,
- PermitRootLogin prohibit-password ogranicza ryzyko brute force,
- PasswordAuthentication yes zwiększa ryzyko brute force,
- PubKeyAuthentication yes jest ustawieniem bezpiecznym i zalecanym, 
- PermitEmptyPassword no jest najlepszym możliwym ustawieniem,
- Brak konfiguracji AllowUsers / DenyUsers oraz AllowGroups / DenyGroups oznacza brak ograniczeń dostępu, 
- MaxAuthTries większa liczba prób zwiększa okno dla brute force, szczególnie bez dodatkowych mechanizmów (m. fail2ban /PAM),
- UsePam yes, oznacza że SSH korzysta z mechanizmów PAM. 

## Najczęstsze misconfigi SSH
- PermitRootLogin yes - zawsze źle na publicznym serwerze,
- PasswordAuthentication yes - umożliwia brute force, 
- brak AllowUsers - każdy może próbować loginów, 
- bark limitów /fail2ban - brak automatycznej obrony,
- PermitEmptyPassword yes - ustawienie pozwalające na pominięcie hasła.

## Kontekst SOC
Zdarzenia związane s SSH takie jak zmiany konfiguracji, uruchamianie usługi oraz wielokrotne nieudane próby logowania, powinny być monitorowane w kontekście bezpieczeństwa. 
- monitorowanie logów (auth.log, journalctl) pod kątem prób brute force oraz nieautoryzowanych zmian konfiguracji,
- weryfikacja efektywnej konfiguracji SSH (sudo sshd -T) po wprowadzeniu zmian.

## Analiza konfiguracji SSH (Case Study)

### Case Study 1

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

Analiza:  
- ustawiony port jest dobrze znany atakującym, często skanowany,
- nasłuch na wszystkich interfejsach IPv4(0.0.0.0). Jeśli serwer posiada wiele interfejsów sieciowych ograniczenie nasłuchu do konkretnego zakresu IP może zmniejszyć powierzchnię ataku.
- PermitRootLogin yes to zawsze niebezpieczny wybór, zwłaszcza na serwerach publicznych,
- PasswordAuthentication yes - opcja pozawala użytkownikom na logowanie za pomocą hasła. W środowiskach produkcyjnych nie jest to zalecane, ponieważ zwiększa ryzyko brute force oraz wycieku lub złamania hasła, 
- PubKeyAuthentication no, wyłączenie opcji logowania kluczem jest niebezpieczne, ponieważ wtedy pozostaje tylko opcja logowania hasłem, co nie jest najbezpieczniejszym sposobem, 
- AllowUsers alice bob - to dobra praktyką, by ograniczać dostęp tylko dla określonej grupy użytkowników, 
- MaxAuthTries 10 - to zbyt duża wartość, zwiększa ryzyko brute force,
- ClientAliveInterval 300, to rozsądne ustawienie, oznacza, że serwer co 5 minut sprawdza czy klient jest nadal aktywny,
- ClientAliveCountMax 3, to poprawne ustawienie, ponieważ jeśli klient nie odpowie 3 razy, to biorąc pod uwagę opcję zapisaną powyżej, serwer przerwie połączenie po 15 minutach.  

Sugerowane działania:
- zmienić port na mniej oczywisty, warto używać zakresu 49152 - 65535 żeby uniknąć kolizji i zmniejszyć ryzyko skanowania, 
- opcjonalnie, celem zwiększenia bezpieczeństwa ograniczyć nasłuch do określonego IP,
- zmienić PermitRootLogin na `no`, zwłaszcza jeśli serwer jest publiczny,
- zmienić autoryzację hasłem na `no` oraz autoryzację kluczem na `yes`. Klucze są znacznie bezpieczniejsze i bardziej odporne na brute force,
- zmienić dozwoloną ilość prób logowania na 3, to dobry kompromis pomiędzy ochroną przed brute force a możliwością pomyłki przy wpisywaniu hasła przez użytkownika.

### Case Study 2

Konfiguracja:  
	port 2222  
	PermitRootLogin prohibit-password  
	PasswordAuthentication no  
	PubKeyAuthentication yes  
	AllowUsers admin  
	MaxAuthTries 6  

Analiza: 
-  port 2222 zmniejsza liczbę automatycznych skanów i prób logowania na domyślnym porcie 22, jedno nie stanowi rzeczywistego mechanizmu bezpieczeństwa,
- AllowUsers - ustawienie dostępu tylko dla jednego konta:
	- zmniejsza powierzchnię ataku, 
	- zmniejsza liczbę potencjalnych punktów kompromitacji,
	- zmniejsza szum informacyjny w logach. 
	Jednak są tutaj też negatywne skutki, ponieważ gdy nastąpi kompromitacja klucza lub konta admina, to atakujący ma pełny dostęp do serwer, bo nie ma żadnego innego konta kontrolnego, natomiast jeśli:
		- klucz admina zostanie uszkodzony, niepoprawnie zostanie zdefiniowane AllowUsers, zmieni się konfiguracja SSH, to serwer jest zablokowany administracyjnie, 
		- tego jednego konta używa kilku adminów, utrudniony jest audyt, nie wiadomo, który admin jakie procesy uruchomił. 
		Dlatego w tym przypadku zawsze warto ustawić dostęp dla grupy adminów, dzięki czemu można uniknąć powyższych konsekwencji.
- PermitRootLogin AllowUsers jest sprawdzane wcześniej niż PermitRootLogin, dlatego root i tak nie uzyska dostępu,
- MaxAuthTries - ustawiona wartość jest zbyt duża, może zwiększyć ryzyko brute force.  

Sugerowane działania:  
- ustawić więcej kont, celem zabezpieczenia dostępu do serwera w razie przejęcia konta, uszkodzenia konfiguracji SSH lub klucza, a także dla ułatwienia audytu,
- zmienić opcję PermitRootLogin na `no`, ponieważ gdy jest ustawiona grupa AllowUsers, w której roota nie ma, jest ona niepotrzebne i nie ma wpływu na nic. Jednak nadal, gdyby ustawienia AllowUses się w przyszłości zmieniły, warto ta linią zabezpieczyć serwer dodatkowo,
- zmienić MaxAuthTries na wartość 3, co zwiększy bezpieczeństwo przed brute force, ale jednocześnie uwzględnia pomyłkę w haśle przez użytkownika. Dodatkowo liczba prób logowania powinna być monitorowana w logach systemowych (np. auth.log), ponieważ duża liczba nieudanych prób może wskazywać na atak brute force. 

### Case Study 3

Konfiguracja:  
	Zakładamy środowisko produkcyjne z SSH:  
	Port 2222  
	PermitRootLogin no  
	PasswordAuthentication no   
	PubKeyAuthentication yes  
	AllowUsers admin  
	MaxAuthTries 3   
Celem jest monitorowanie logów auth.log lub journalctl w poszukiwaniu nieprawidłowych zachowań brute force i potencjalnych kompromitacji.  

Log 1  
mar 6 09:15:23 kali sshd [2345]: Accepted publickey for admin from 192.168.1.10 port 54321 ssh2   

log 2  
mar 6 03:42:11 kali sshd [2350]: Failed password for admin from 185.203.56.77 port 42213 ssh2  
mar 6 03:42:11 kali sshd [2350]: Failed password for admin from 185.203.56.77 port 42213 ssh2  
mar 6 03:42:11 kali sshd [2350]: Failed password for admin from 185.203.56.77 port 42213 ssh2  
mar 6 03:42:11 kali sshd [2350]: Failed password for admin from 185.203.56.77 port 42213 ssh2  

Log 3  
mar 6 22:17:02 kali sshd [2367]: Accepted publickey for admin from 198.51.100.45 port 42213 ssh2: RSA SHA256:abcd1234...   

Analiza:  
- konfiguracja SSH jest ustawiona poprawnie, z wyjątkiem AllowUsers - ma tylko jednego użytkownika, co zwiększa ryzyko administracyjnego zablokowania serwera lub utraty kontroli nad serwerem,
- log 1 - logowanie z systemu kali za pomocą klucza dla użytkownika admin, odbywa się z lokalnego adresu IP, port 54321. Tutaj nic nie wskazuje na to, że dzieje się coś złego. Wygląda na zwyczajne logowanie admina w godzinach pracy z sieci lokalnej,
- log 2 - wiele zakończonych niepowodzeniem prób logowania do konta admin za pomocą hasła co jest zabronione w konfiguracji, logowania odbywają się późno w nocy co 4 sekundy z zewnętrznego adresu IP. Wzorzec wskazuje na zautomatyzowaną próbę brute force. Jeśli PasswordAutenthication=no, a w logach pojawiają się 'Failed password', może to oznaczać także: 
	- próbę ataki zanim konfiguracja została zastosowana,
	- inny mechanizm (np. PAM / failback),
	- lub próbę nieobsługiwanej metody uwierzytelniania. 
- log 3 - logowanie po 22 wieczorem powinno budzić podejrzenia, zwłaszcza jeśli to nie są godziny pracy. Teoretycznie to mógł być admin pragnący przeprowadzić jakieś prace na serwerze poza godzinami pracy, to jeszcze nie jest powód do paniki, jednak przypadek wymaga dokładnego sprawdzenia, ponieważ może to być kompromitacja klucza lub anomalne logowanie (czas + IP).   

Sugerowane działania:
- dodać jedno lub dwa konta do AllowUsers,
- sprawdzić zabezpieczenia konfiguracji, upewnić się, że MaxAuthTries ma niską wartość, że hasło nie jest dozwolone podczas logowania, czy logowanie root jest dozwolone oraz jaka jest grupa AllowUsers,
- sprawdzić dokładnie dane z logu 3 - czy IP jest zaufanym adresem, upewnić się czy klucz nie wyciekł, sprawdzić w logach jakie działania przeprowadził admin po zalogowaniu i jak długo trwała jego sesja, skonsultować logi z adminem, celem potwierdzenia, że rzeczywiście się logował w tym czasie. Ponadto należy sprawdzić fingerprint klucza, czy zgadza się z faktycznym. 
