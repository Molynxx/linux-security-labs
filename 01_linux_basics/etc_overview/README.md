# Lab: /etc - analiza plików konfigruacyjnych systemu

## Cel
Celem tego laboratorium było zapoznanie się z plikami konfiguracyjnymi w katalogu '/etc' które mają znaczenie dla bezpieczeństwa systemu, w szczególności:
- '/etc/passwd'
- '/etc/group'
- '/etc/shadow'
- '/etc/sudoers'
- '/etc/ssh/sshd_config'

Analiza tych plików pozwala zrozumieć:
- jakie konta istnieją w systemie, 
- które mają dostęp interaktywny,
- jakie grupy i uprawnienia mają użytkownicy,
- które konta mogą stanowić potencjalne zagrożenie

## /etc/passwd
 
### Czym jest /etc/passwd
Plik '/etc/passwd' zawiera informacje o użytkownikach systemu:
- nazwa użytkownika, 
- hash hasła lub marker 'x',
- UID (User ID),
- GID (Group ID),
- katalog domowy,
- powłoka logowania (shell).

### Wykonane kroki
- Przejrzano plik poleceniem cat /etc/passwd,
- Sprawdzono ostatnie logowania użytkowników za pomocą polecenia last

### Obserwacje
- Występują trzy rodzaje użytkowników:
	- root - konto administracyjne z dostępem interaktywnym,
	- użytkownicy systemowi - UID < 1000, powłoka /user/bin/nologin lub /bin/false,
	- użytkownicy interaktywni - UID > 1000, powłoka /bin/bash, /bin/zsh lub /bin/sh, katalog domowy w /home.
- Niektóre konta systemowe mogą mieć interaktywny shell (np. postgres) - to są wyjątki.

### Wnioski bezpieczeństwa
- Każdy użytkownik systemowy z możliwością interaktywnego logowania wymaga weryfikacji.
- Należy sprawdzić:
	- czy konto zostało zgłoszone i utworzone w sposób zgodny z polityką, 
	- kiedy zostało założone,
	- kiedy i skąd logowało się konto (last, ssh, lokalnie),
	- czy logowania były w nietypowych godzinach.
- Użytkownicy z UID >= 1000 powinni mieć powłokę interaktywną i katalog domowy; każde odstępstwo wymaga uwagi.

## /etc/group

### Czym jest /etc/group
Plik /etc/group zawiera informacje o grupach w systemie:
- nazwa grupy, 
- hash hasła lub marker 'x',
- GID grupy,
- członkowie grupy.

### Wykonane kroki
- Przejrzano plik poleceniem cat /etc/group,
- Użyto grep sudo dla pliku, aby sprawdzić którzy użytkownicy należą do grupy sudo,
- Użyto grep -E 'sudo|adm|docker' /etc/group, aby sprawdzić użytkowników należących do uprzywilejowanych grup,
- Celem powyższych działań było upewnienie się, że żaden nieautoryzowany użytkownik nie uzyskał dostępu,
- Utworzono katalog na skrypty i ustawiono jego uprawnienia wyłącznie dla właściciela,
- Stworzono skrypt wyświetlający wszystkich użytkowników z /etc/passwd oraz listę grup, do których należą,
- Dodano skrypt do zmiennej środowiskowej $PATH, aby można go było uruchomić z dowolnego miejsca w systemie.

### Napotkane problemy
- Polecenie source ~/.bashrc zgłaszało wiele błędów, a skrypt .sh w $PATH nie działał.

### Rozwiązanie problemu
- Zrozumiano, jak dziłają powłoki w systemie,
- Kali Linux używa powłoki zsh, a nie bash, co było przyczyną problemów, 
- Aby naprawić problem, skrypt .sh przeniesiono do pliku konfiguracyjnego powłoki zsh, tj. ~/.zshrc,
- Po tych krokach problem został rozwiązany i skrypt działał poprawnie.

### Obserwacje
- Podczas modyfikacji zmiennych środowiskowych należy zwracać uwagę na używaną powłokę systemu,
- W pliku /etc/group wyróżniono dwa rodzaje grup:
	- Grupy systemowe - zawierają użytkowników systemowych, GID <1000, często nie pokazują użytkowników,
	- Grupy użytkowników - zawierają użytkowników interaktywnych, GID >= 1000, członkowie są wymienieni w pliku.

### Wnioski bezpieczeństwa
- Jeśli grupa systemowa nie zawiera żadnego użytkownika, jest to normalne i można zignorować,
- Jeśli grupa systemowa zawiera użytkownika interaktywnego, należy sprawdzić, czy powinien mieć w niej dostęp - szczególnie w grupach sudo, adm, docker, 
- Grupy użytkowników interaktywnych należy zawsze dokładnie sprawdzić, aby upewnić się, że nie ma nieautoryzowanych członków. 

## /ect/shadow

### Czym jest /etc/shadow
Plik /etc/shadow zawiera informacje o:
- blokadach kont,
- danych dotyczących zmiany hasła, 
- wymuszeniu zmiany hasła,
- wygaśnięciu konta.

Szczegółowe informacje w każdej linii /etc/shadow (oddzielone dwukropkiem : ):
- login użytkownika,
- hash hasła,
- ostatnia zmiana hasła (dni od 1970 roku),
- minimalna liczba dni między zmianami hasła,
- maksymalna liczba dni ważności hasła,
- ostrzeżenie o wygaśnięciu hasła - ile dni przed wygaśnięciem hasła system informuje użytkownika o konieczności zmiany hasła,
- nieaktywność  - ile dni po wygaśnięciu hasła konto zostaje zablokowane, 
- ostatnie pole - rzadko używane, zwykle puste.

### Wykonane kroki
- Sprawdzenie uprawnień pliku /etc/shadow za pomocą ls -l, aby upewnić się, że pełne prawa ma tylko root, grupa ma prawo do odczytu (opcjonalnie), others nie mają żadnych praw. 
- Przejrzenie pliku /etc/shadow za pomocą polecenia cat.
- Analiza informacji i zrozumienie ich znaczenia. 

### Obserwacje
- W pliku można wyróżnić dwa rodzaje kont:
	- Użytkownicy systemowi - UID < 1000, często mają zablokowaną możliwość logowania ( !, !!, !* ).
	- Użytkownicy interaktywni - UID >=1000, posiadają hash hasła. 

### Wnioski bezpieczeństwa
- Na co zwrócić uwagę w pliku /etc/shadow:
	- czy konto systemowe lub interaktywny użytkownik ma hasło czy jest zablokowane ( !,!!,!* ), 
	- czy konto interaktywne ma wymuszoną zmianę hasła,
	- czy ustawione są ostrzeżenia przed wygaśnięciem hasła,

### Potencjalne zagrożenia
- Konta systemowe z aktywnym hashem zamiast blokady (!, !!) - może to świadczyć o nieautoryzowanym dostępie. Należy wtedy sprawdzić w pliku /etc/passwd czy konto ma dostęp do /bin/bash. To jednak nie jest równoznaczne z atakiem, zdarza się, że administrator tworzy konto systemowe z hasłem dla specyficznych usług. SOC powinien sprawdzić kontekst konta. 
- jeśli tak to zachodzi podejrzenie backdoora. 
- Konta systemowe z czasową blokadą logowania (!) mogą zostać odblokowane przez osobę nieuprawnioną, tutaj również należy sprawdzić w /etc/passwd czy konto ma dostęp do /bin/bash. Jeśli tak to potencjalne zagrożenie. 
- Konta interaktywnych użytkowników, które nie mają wymuszonej zmiany hasła lub nie mają go wcale i mają bardzo długie maksymalne dni ważności również stanowią potencjalne zagrożenie. 

### Przykłady zabezpieczeń dla kont podejrzanych 
- Zablokowanie interaktywnego shella:
	sudo usermod -s /usr/sbin/nologin <nazwa_użytkownika>
- Trwałe zablokowanie hasła:
	sudo passwd -l <nazwa_użytkownika>
- Ograniczenie dostępu do SSH - w pliku /etc/ssh/sshd_config uwstawić:
	DenyUsers <nazwa_użytkownika>
- Sprawdzenie ustawień sudo:
	sudo -l -U <nazwa_użytkownika>
- monitorowanie zmian w shadow:
	stat /etc/shadow
	zwracamy tutaj uwagę na godzinę zmian, zmianę właściciela, zmianę zawartości pliku, zmiany uprawnień pliku.

## /etc/sudoers

### Czym jest /etc/sudoers
Plik /etc/sudoers definiuje kto i jakie polecenia może uruchamiać z podwyższonymi uprawnieniami. Obejmuje dane:
- kto,
- na jakim hoście,
- jako jaki użytkownik,
- jakie polecenie może uruchomić.

### Wykonane kroki
- Przejrzano plik /etc/sudoers za pomocą polecenia sudo less,
- Przejrzano pliki znajdujące się w katalogu /etc/sudoers.d/* za pomocą polecenia sudo ls,
- Uruchomiono sudo visudo żeby otworzyć plik /etc/sudoers z możliwością edycji,
- Utworzono nowego użytkownika nienależącego do żadnej uprzywilejowanej grupy,
- Utworzono plik w /etc/sudoers.d/ o nazwie zgodnej z nazwą użytkownika za pomocą polecenia sudo visudo -f /etc/sudoers.d/user-apt,
- W powyższym pliku nadano user uprawnienia root dla sudo apt update i sudo apt upgrade wpisując linię pliku 'user ALL=(root) /usr/bin/apt update, /usr/bin/apt upgrade',
- Sprawdzono jakie polecenia z uprawnieniami root może uruchomić user, za pomocą polecenia sudo -l -U user.

### Obserwacje 
- Grupa root ma pełne uprawnienia bez użycia sudo, natomiast użytkownicy z grupy sudo uzyskują pełne uprawnienia za pomocą polecenia sudo, zgodnie z konfiguracją pliku /etc/sudoers,
- Powyższe oznacza, że każdy użytkownik dodany do jednej z tych grup zyska pełne prawa do wszystkiego jako root,
- Plik /etc/sudoers jest chroniony uprawnieniami i nie może być odczytany przez zwykłego użytkownika. Do bezpiecznej edycji używa się wyłącznie visudo,
- Plik /etc/sudoers można otworzyć bezpiecznie poleceniem sudo less, ponieważ używając tego polecenia nie ma możliwości edycji pliku,
- W katalogu /etc/sudoers.d/* znadują się pliki konfiguracyjne uprawnień dla poszczególnych użytkowników, m.in. plik zdefiniowany przeze mnie dla nowego użytkownika user.

### Wnioski bezpieczeństwa
Na co trzeba zwrócić uwagę:
- Kto ma pełny dostęp, w pliku /etc/sudoers widać jakie prawa mają grupy root i sudo, sprawdzić w /etc/group kto należy do tych grup i czy na pewno powinien tam należeć, 
- Kto ma NOPASSWD,
- Czy są nietypowe wpisy w pliki /etc/sudoers,
- Czy są nietypowe, nieznane pliki w katalogu /etc/sudoers.d/

### Potencjalne zagrożenia
- Każdy użytkownik z NOPASSWD to potencjalny wektor ataku, 
- Nietypowe polecenia pozwalające na zapis do systemowych katalogów (/etc, /root, var) powinny być dokładnie sprawdzone, czy rzeczywiście powinny tam być, 
- Jeśli użytkownik jest w grupie sudo, dla której konfiguracja pliku /etc/sudoers nadaje pełne uprawnienia, nie ma możliwości zablokowania mu czegokolwiek, dlatego jeśli użytkownik ma mieć tylko niektóre prawa roota należy usunąć go z grupy sudo i dodać stosowny plik w /etc/sudoers.d/ by nadać mu indywidualne uptrawenienia,
- Chcąc zabronić użytkownikowi dostępu  do /bin/bash ze względów bezpieczeństwa, nie można zrobić tego dodając wpis w /etc/sudoers.d/plikusera 'user ALL = (ALL) ALL !/bin/bash', ponieważ można to obejść korzystając z innych powłok niż bash, np użycie podpowłoki zsh,
- Nie zadziała również zapis 'user ALL =(ALL) ALL !/bin/bash, !/bin/sh, !/bin/zsh', ponieważ nadal można to obejść na kilka sposobów:
	- sudo -s,
	- sudo -i,
	- env.
- Najskuteczniejszym sposobem zabezpieczenia jest zapis w osobnym pliku dla danego użytkownika w /etc/sudoers.d/, gdzie można przykładowo nadać uprawnienia root użytkownikowi dla poleceń apt update i apt upgrade za pomocą zapisu w pliku : 'user ALL=(root) /usr/bin/apt update, /usr/bin/apt upgrade.


## /etc/ssh

### Czym jest /etc/ssh
Jest to katalog przechowujący pliki konfiguracyjne dla usługi SSH (Secude Shell), służącej do zdalnego logowania do systemu. Główny plik konfiguracyjny serwera SSH znajduje się w ścieżce:  /etc/ssh/sshd_config. W pliku można zdefiniować m.in.:
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
	- listenaddress,
	- addressfamily,
	- permiotrootlogin,
	- passwordauthentication,
	- pubkeyauthentication,
	- allowusers,
	- denyusers,
	- allowgroups,
	- denygroups, 
	- maxauthtries,
	- usepam.
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
	- listenaddress ustawiony na 0.0.0.0 oraz [::], co oznacza nasłuch na wszystkich adresach IPv4 i IPv6,
	- permitrootloigin ustawione jest na prohibit-password,
	- passwordauthentication ustawiony jest na yes,
	- pubkeyauthentication ustawiony jest na yes,
	- permitepmtypassword ustawiony jest na no, 
	- allowuserws oraz denyusers nie są skonfigurowane,
	- allowgropups oraz denygroups nie są skonfigurowane, 
	- maxauthtries ustawione jest na 6,
	- usepam ustawione jest na yes.
- Pomimo ustawienia maxauthtries na wartość 6, po trzech nieudanych próbach logowania sesja SSH została przerwana, co wskazuje na działanie  polityk PAM.

### Wnioski bezpieczeństwa
- Domyślny port 22 jest powszechnie znany i często skanowany. Zmiana portu na niestandardowy (niekolidujący z innymi usługami) może ograniczyć automatyczne ataki typu brute force,
- Nasłuchiwanie na wszystkich adresach IPv6 ([::}) nie zawsze jest konieczne. Jeśli nIPv6 nie jest używane, warto rozważyć ustawienie familyaddress na inet, aby ograniczyć nasłuch wyłącznie do IPv4. 
- Nasłuchiwanie na 0.0.0.0 jest poprawne funkcjonalnie, jednak z punktu widzenia bezpieczeństwa można ograniczyć powierzchnię ataku poprzez przypisanie SSH do konkretnego interfejsu lub adresu IP,
- Ustawienie permitrootlogin prohibit-password jest dobrą praktyką uniemożliwia logowanie roota za pomocą hasła, co znacząco ogranicza ryzyko brute force,
- passwordauthentication yes stanowi potencjalne zagrożenie bezpieczeństwa, bezpieczniejszym rozwiązaniem jest wyłączenie logowania hasłem i korzystanie wyłącznie z kluczy SSH.
- pubkeyauthentication yes jest ustawieniem bezpiecznym i zalecanym, 
- permitemptypassword no jest najlepszym możliwym ustawieniem - logowanie bez hasła znacząco zwiększa powierzchnię ataku, 
- Brak konfiguracji allowusers / denyusers oraz allowgroups / denygroups oznacza brak ograniczeń dostępu. Najbezpieczniejszym podejściem jest jawne wskazanie użytkowników lub grup, które mogą korzystać z AAH, pamiętając, że reguły deny* mają pierwszeństwo,
- maxauthtries ustawione na 6 może ułatwić brute force, zalecaną wartością jest 3, z uwzględnieniem możliwości pomyłki użytkownika,
- usepam yes, oznacza że SSH korzysta z mechanizmów PAM. SSH korzysta z PAM, dlatego ograniczenia logowania mogą wynikać z polityk PAM, a nie tylko z ustawień SSH. W tym przypadki to właśnie PAM ograniczył liczbę prób logowania do 3, mimo wyższej wartości w konfiguracji SSH, co należy uznać za poprawne działanie mechanizmów bezpieczeństwa.

### Kontekst SOC
Zmiany w konfiguracji SSH, uruchamianie lub zatrzymywanie usługi SSH oraz wielokrotnie nieudane próby logowania są zdarzeniami istotnymi z punktu widzenie SOC. Powinny one być monitorowane i korelowane z logami systemowymi (journalctl, auth.log) oraz alertami dotyczącymi brute force, nieautoryzowanego dostępu lub nieautoryzowanych zmian konfiguracyjnych.
