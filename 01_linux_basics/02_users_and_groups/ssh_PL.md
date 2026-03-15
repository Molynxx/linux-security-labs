## /etc/ssh

### Czym jest /etc/ssh
Jest to katalog przechowujący pliki konfiguracyjne dla usługi SSH (Secure Shell), służącej do zdalnego logowania do systemu. Główny plik konfiguracyjny serwera SSH znajduje się w ścieżce:  /etc/ssh/sshd_config. W pliku można zdefiniować m.in.:
- sposób łączenia się do systemów zdalnego,
- port i adresy nasłuchu,
- metody uwierzytelniania,
- uprawnienia użytkowników i grup.
Linie rozpoczynające się znakiem # są komentarzami. Zawierają one informacje pomocnicze oraz przykładowe opcje, które nie są aktywne, dopóki nie zostaną odkomentowane lub nadpisane. Odkomentowanie lub dodanie opcji powoduje zastąpienie wartości domyślnej. 
Dobrą praktyką administracyjną jest dodawanie własnych ustawień na końcu pliku, zamiast edytowania istniejących komentarzy - ułatwia to audyt i analizę zmian.
Warto podkreślić, że plik sshd_config nie pokazuje domyślnych ani aktualnie obowiązujących ustawień SSH. Aby sprawdzić rzeczywistą (efektywną) konfigurację, należy użyć polecenia: sudo sshd -T. POlecenie to wyświetla efektywną konfigurację SSH, czyli wartości domyślne połączone z tym, co zostało nadpisane w plikach konfiguracyjnych. 

### Wykonane kroki
- Przejrzano plik /etc/ssh/sshd_config,
- Sprawdzono efektywną konfigurację SSH za pomocą polecenia sudo sshd -T,
- Za pomocą polecenia sudo sshd -T | grep "słowo kluczowe" sprawdzono konfigurację:
	- port,
	- ListenAddress,
	- AddressFamily,
	- PermitRootLogin,
	- PasswordAuthentication,
	- PubKeyAuthentication,
	- AllowUsers,
	- DenyUsers,
	- AllowGroups,
	- DenyGroups, 
	- MaxAuthTries,
	- UsePam.
- Sprawdzono zawartość katalogu /etc/security/, w którym nie znaleziono pliku pwquality.conf,
- Zainstalowano pakiet libpam-pwquality za pomocą polecenia sudo apt install,
- Sprawdzono plik /etc/pam.d/common-password,
- Sprawdzono status usługi SSH za pomocą polecenia sudo systemctl status ssh,
- Włączono usługę SSH poleceniem sudo sytemctl start ssh,
- Zweryfikowano nasłuch usługi za pomocą polecenia sudo ss -tulpn | grep ssh,
- Wyłączono usługę SSH poleceniem sudo systemctl stop ssh.

### Obserwacje
- W pliku /etc/ssh/sshd_config wszystkie linie są zakomentowane, co oznacza, że nie wprowadzono niestandardowych ustawień, a SSH działa na konfiguracji domyślnej,
- Usługa SSH była domyślnie wyłączona w systemie Kali Linux,
- Po sprawdzeniu efektywnej konfiguracji poleceniem sudo sshd -T stwierdzono, że:
	- port ustawiony jest na 22, 
	- ListenAddress ustawiony na 0.0.0.0 oraz [::], co oznacza nasłuch na wszystkich adresach IPv4 i IPv6,
	- PermitRootLoigin ustawione jest na prohibit-password,
	- PasswordAuthentication ustawiony jest na yes,
	- PubKeyAuthentication ustawiony jest na yes,
	- PermitEpmtyPassword ustawiony jest na no, 
	- AllowUsers oraz DenyUsers nie są skonfigurowane,
	- AllowGropups oraz DenyGroups nie są skonfigurowane, 
	- MaxAuthTries ustawione jest na 6,
	- UsePam ustawione jest na yes.
- Pomimo ustawienia MaxAuthTries na wartość 6, po trzech nieudanych próbach logowania sesja SSH została przerwana, co wskazuje na działanie  polityk PAM.

### Wnioski bezpieczeństwa
- Domyślny port 22 jest powszechnie znany i często skanowany. Zmiana portu na niestandardowy (niekolidujący z innymi usługami) może ograniczyć automatyczne ataki typu brute force,
- Nasłuchiwanie na wszystkich adresach IPv6 ([::}) nie zawsze jest konieczne. Jeśli nIPv6 nie jest używane, warto rozważyć ustawienie AddressFamily na inet, aby ograniczyć nasłuch wyłącznie do IPv4. 
- Nasłuchiwanie na 0.0.0.0 jest poprawne funkcjonalnie, jednak z punktu widzenia bezpieczeństwa można ograniczyć powierzchnię ataku poprzez przypisanie SSH do konkretnego interfejsu lub adresu IP,
- Ustawienie PermitRootLogin prohibit-password jest dobrą praktyką uniemożliwia logowanie roota za pomocą hasła, co znacząco ogranicza ryzyko brute force,
- PasswordAuthentication yes stanowi potencjalne zagrożenie bezpieczeństwa, bezpieczniejszym rozwiązaniem jest wyłączenie logowania hasłem i korzystanie wyłącznie z kluczy SSH.
- PubKeyAuthentication yes jest ustawieniem bezpiecznym i zalecanym, 
- PermitEmptyPassword no jest najlepszym możliwym ustawieniem - logowanie bez hasła znacząco zwiększa powierzchnię ataku, 
- Brak konfiguracji AllowUsers / DenyUsers oraz AllowGroups / DenyGroups oznacza brak ograniczeń dostępu. Najbezpieczniejszym podejściem jest jawne wskazanie użytkowników lub grup, które mogą korzystać z SSH, pamiętając, że reguły deny* mają pierwszeństwo,
- MaxAuthTries ustawione na 6 może ułatwić brute force, zalecaną wartością jest 3, z uwzględnieniem możliwości pomyłki użytkownika,
- UsePam yes, oznacza że SSH korzysta z mechanizmów PAM. SSH korzysta z PAM, dlatego ograniczenia logowania mogą wynikać z polityk PAM, a nie tylko z ustawień SSH. W tym przypadki to właśnie PAM ograniczył liczbę prób logowania do 3, mimo wyższej wartości w konfiguracji SSH, co należy uznać za poprawne działanie mechanizmów bezpieczeństwa.

### Kontekst SOC
Zmiany w konfiguracji SSH, uruchamianie lub zatrzymywanie usługi SSH oraz wielokrotnie nieudane próby logowania są zdarzeniami istotnymi z punktu widzenie SOC. Powinny one być monitorowane i korelowane z logami systemowymi (journalctl, auth.log) oraz alertami dotyczącymi brute force, nieautoryzowanego dostępu lub nieautoryzowanych zmian konfiguracyjnych.
- Należy monitorować auth.log i jouralctl pod kątem prób brute force oraz nieautoryzowanych zmian w konfiguracji SSH,
- Warto regularnie sprawdzać efektywną konfigurację SSH (sudo ssh -T) po każdej zmianie plików konfiguracyjnych,

