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
	- Hash "*" lub "!" oznacza, że konto jest zablokowane,
- Brak hasła lub ustawienie zbyt długiego maksymalnego okresu hasła (max days)może stanowić ryzyko bezpieczeństwa,
- Aby sprawdzić szczegóły konta pod kątem ważności hasła, można użyć polecenia: chage -l <nazwa_użytkownika>.

### Potencjalne zagrożenia
- Konta systemowe z aktywnym hashem zamiast blokady (!, !!, *) - może to świadczyć o nieautoryzowanym dostępie. Należy wtedy sprawdzić w pliku /etc/passwd czy konto ma dostęp do /bin/bash. To jednak nie jest równoznaczne z atakiem, zdarza się, że administrator tworzy konto systemowe z hasłem dla specyficznych usług. SOC powinien sprawdzić kontekst konta. 
- jeśli tak to zachodzi podejrzenie backdoora. 
- Konta systemowe z czasową blokadą logowania (!) mogą zostać odblokowane przez osobę nieuprawnioną jeśli atakujący uzyskał uprawnienia administracyjne, tutaj również należy sprawdzić w /etc/passwd czy konto ma dostęp do /bin/bash. Jeśli tak to potencjalne zagrożenie. 
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