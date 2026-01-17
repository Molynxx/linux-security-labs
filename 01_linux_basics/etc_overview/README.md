# Lab: /etc - analiza plików konfiguacyjnych systemu

## Cel
Celem tego laboratorium było zapoznanie się z plikami konfiguracyjnymi w katalogu '/etc' które mają znaczenie dla bezpieczeństwa systemu, w szczególności:
- '/etc/passwd'
- '/etc/group'
- '/etc/shadow'
- '/etc/sudores'
- '/etc/shh/sshd_config'

Analiza tych plików pozwala zrozumieć:
- jakie konta istnieją w systemie, 
- które mają dostęp interaktywny,
- jakie grupy i uprawnienia mają użytkownicy,
- które konta mogą stanowić potencjalen zagrożenie

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
- Sprawdzono ostanie logowania użytkowników za pomocą polecenia last

### Obserwacje
- Występują trzy rodzaje użytkowników:
	- root - konto administracyjne z dostępem interaktywnym,
	- użytkownicy systemowi - UID < 1000, powłoka /user/bin/nologin lub /bin/false,
	- użytkownicy interaktywni - UID > 1000, powłoka /bin/bash, /bin/zsh lub /bin/sh, katalog domowy w /home.
- Niektóre konta systemowe mogą mieć interaktywny shell (np. polstgres) - to są wyjątki.

### Wnioski bezpieceństwa
- K  ażdy użytkownik systemowy z możliwością interaktywnego logowania wymaga weryfikacji.
- Należy sprawdzić:
	- czy konto zostało zgłoszone i utworzone w sposób zgodny z polityką, 
	- kiedy zostało założone,
	- kiedy i skąd logowało się konto (last, ssh, lokalnie),
	- czy logowania były w nietypowych godzinach.
- Użytkownicy z UID >= 1000 poqinni mieć powłokę interaktywną i katalog domowy; każde odstępstwo wymaga uwagi.

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
- Użytko grep -E 'sudo|adm|docker' /etc/group, aby sprawdzić użytkowników należących do uprzywilejowanych grup,
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
- W pliku /etc/group wyrózniono dwa rodzaje grup:
	- Grupy systemowe - zawierają użytkownikóœ systemowych, GID <1000, często nie pokazują użytkowników,
	- Grupy użytkowników - zawierają użytkowników interaktywnych, GID >= 1000, członkowie są wymienieni w pliku.

### Wnioski bezpieczeństwa
- Jeśli grupa systemowa nie zawiera żadnego użytkownika, jest to normalne i można zignorować,
- Jeśli grupa systemowa zawiera użytkownika interaktywnego, należy sprawdzić, czy powinen mieć w niej dostęp - szczgónie w grupach sudo, adm, docker, 
- Grupy użytkowników interaktywnych należy zawsze dokładnie sprawdzić, aby upenić się, że nie ma nieautoryzowanych członków. 

## /ect/shadow

### Czym jest /etc/shadow
Plik /etc/shadow zawiera informacje o:
- blokadach kont,
- danych dotyczących zmiany hasłą, 
- wymuszeniu zmiany hasła,
- wygaśnięciu konta.
Szczegółowe informacje w każdej linii /etc/shadow (oddzielone dwukropkiem : ):
- login użytkownika,
- hash hasła,
- ostatnia zmiana hasła (dni od 1970 roku),
- minimalna liczba dni między zmianami hasła,
- maksymalna liczba dni ważniści hasła,
- ostrzeżenie o wygaśnięciu hasła - ile dni przed wygaśnięciem hasła system informuje użytkownika o konieczności zmiany hasła,
- nieaktywność  - ile dni po wygaśnięciu hasła konto zostaje zablokowane, 
- ostanie pole - rzadko używane, zwykle puste.

### Wykonane kroki
- Sprawdzenie uprawnień pliku /etc/shadow za pomocą ls -l, aby upewnić się, że pełne prawa ma tylko root, grupa ma prawo do odczytu (opcjonalnie), others nie mają żadnych praw. 
- Przejrzenie pliku /etc/shadow za pomocą polecenia cat.
- Analiza infromacji i zrozumienie ich znaczenia. 

### Obserwacje
- W pliku można wyróżnić dwa rodzaje kont:
	- Użytkownicy systemowi - UID < 1000, często mają zablokowaną możliwość logoeania (!, !!, !).
	- Użytkownicy interaktywni - UID >=1000, posiadają hask hasła. 

### Wnioski bezpieczeństwa
- Na co zwrócić uwagę w pliku /etc/shadow:
	- czy konto systemowe lub interaktywny użytkownik ma hasło czy jet zablokowane ( !,!!,!* ), 
	- czy konto interaktywne ma wymuszoną zmianę hasła,
	- czy ustawione są ostrzeżenia przed wygaśnięciem hasła,

### Potencjalne zarożenia
- Konta systemowe z aktywnym hashem zamist blokady (!, !!) - może to świadczyć o nieautoryzowanym dostępie. Należy wtedy sprawdzić w pliku /etc/passwd czy konto ma dostęp do /bin/bash - jeśli tak to zachodzi podejrzenie backdora. 
- Konta systemowe z czasową blokadą logownia (!) moga zostać odblokowane przez osobę nieuprawnioną, tutaj również należy sprawdzić w /etc/passwd czy konto ma dostep do /bin/bash. Jeśli tak to potencjalne zagrożenie. 
- Konta interaktywnych użytkowników, które nie mają wymuszonej zmiany hasła lub nie mają go wcale i mają bardzo długie maksymalne dni ważności również stanowią potencjalne zagrożenie. 

### Przykłady zabezpieczeń dla kont podejrzanych 
- Zablokowanie interaktywnego shella:
	sudo usermod -s /usr/sbin/nologin <nazwa_użytkownika>
- Trwałe zablokowanie hasła:
	sudo passwd -l <nazwa_użytkownika>
- Ograniczenie dostępu do SSH - w pliku /etc/ssh/sshd_config uwstawić:
	DebtUsers <nazwa_użytkownika>
- Sprawdzenie ustawień sudo:
	sudo -l -U <nazwa_użytkownika>
- monitorowanie zmian w shadow:
	stat /etc/shadow
	zwracamy tutaj uwagę na godzinę zmian, zmianę właściciela, zmianę zawartości pliku, zmiany uprawnień pliku.