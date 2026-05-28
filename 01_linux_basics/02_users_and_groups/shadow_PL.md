# /etc/shadow

## Cel laboratorium: analiza wpisów w pliku `/etc/shadow`

## Czym jest /etc/shadow
Plik przechowuje hash haseł i politykę ich ważności.
Szczegółowe informacje w każdej linii /etc/shadow (oddzielone dwukropkiem `:`):
- login użytkownika,
- hash hasła,
- lastchg - ostatnia zmiana hasła (dni od 1970 roku),
- min - minimalna liczba dni między zmianami hasła,
- max - maksymalna liczba dni ważności hasła,
- warn - ostrzeżenie o wygaśnięciu hasła,
- inactive - nieaktywność  - ile dni po wygaśnięciu hasła konto zostaje zablokowane, 
- reserved - ostatnie pole - rzadko używane, zwykle puste.

## Obserwacje
- w pliku można wyróżnić trzy rodzaje kont:
	- użytkownicy systemowi - często zablokowana możliwość logowania:
		- ! przed hashem - hasło zablokowane (login hasłem niemożliwy),
		- !! zwykle oznacza konto utworzone bez ustawionego hasła (np. useradd bez passwd),
		- \* brak możliwości logowania hasłem,
		- puste pole - brak hasła, krytyczne ryzyko,
	- użytkownicy interaktywni - posiadają hash hasła,
	- root - zazwyczaj posiada hash hasła, blokada `!` występuje tylko w specyficznych konfiguracjach, np. systemy z wyłączonym logowaniem root,
	Aby sprawdzić na podstawie UID, które konto jest systemowe, a które interaktywne trzeba skorelować działania z plikiem `/etc/passwd`.


## Wnioski bezpieczeństwa
- Na co zwrócić uwagę w pliku `/etc/shadow`:
	- czy konto systemowe lub interaktywny użytkownik ma hasło czy jest zablokowane (!,!!,!*), 
	- czy konto interaktywne ma wymuszoną zmianę hasła,
	- czy ustawione są ostrzeżenia przed wygaśnięciem hasła,
	- czy jest ustawiona ilość dni do zablokowania konta po wygaśnięciu hasła,
	- jaki jest użyty algorytm hashujący,
- Brak hasła lub ustawienie zbyt długiego maksymalnego okresu hasła (max days)może stanowić ryzyko bezpieczeństwa,
- Aby sprawdzić szczegóły konta pod kątem ważności hasła, można użyć polecenia: chage -l <nazwa_użytkownika>, 
- słaby algorytm hashujący może być podatny na brute force, najlepszym wyborem są algorytmy yescrypt i bcrypt. To wolne algorytmy, specjalnie zaprojektowane do przechowywania haseł i są odporne na ataki brute force offline dzięki swojej kosztowności obliczeniowej. Algorytm MD5 zawsze powinien zaalarmować SOC/IR.

## Potencjalne zagrożenia
- Konta systemowe z aktywnym hashem zamiast blokady (!, !!, *) wymaga weryfikacji przeznaczenia i uprawnień. Należy sprawdzić w pliku `/etc/passwd` czy konto ma dostęp do `/bin/bash`. To nie musi oznaczać ataku, możliwe scenariusze:
	- konto serwisowe,
	- konto aplikacyjne,
	- migracja systemu,
	- konfiguracja legacy (odziedziczone, stare, historyczne rozwiązanie).
	IR zawsze bada kontekst:
	- kto stworzył konto,
	- kiedy,
	- czy jest w sudo,
	- czy logowało się przez SSH,
	- czy ma klucz w ~/.ssh.
- konta systemowe z czasową blokadą logowania `!` mogą zostać odblokowane przez osobę nieuprawnioną jeśli atakujący uzyskał uprawnienia administracyjne - należy sprawdzić w `/etc/passwd` czy konto ma dostęp do powłoki interaktywnej. Jeśli tak to potencjalne zagrożenie,
- konta interaktywnych użytkowników, które nie mają wymuszonej zmiany hasła lub nie mają go wcale, mają długie maksymalne okresy ważności hasła również stanowią potencjalne zagrożenie. 


## Przykłady zabezpieczeń dla kont podejrzanych 
- Zablokowanie interaktywnego shella: `sudo usermod -s /usr/sbin/nologin <nazwa_użytkownika>`,
- Trwałe zablokowanie hasła: `sudo passwd -l <nazwa_użytkownika>`,
- Ograniczenie dostępu do SSH - w pliku /etc/ssh/sshd_config ustawić: `DenyUsers <nazwa_użytkownika>`,
- Sprawdzenie ustawień sudo: `sudo -l -U <nazwa_użytkownika>`
- monitorowanie zmian w shadow:
	- ręcznie: `stat /etc/shadow` (godzin zmian, zmiana właściciela, zmiana zawartości pliku, zmiany uprawnień pliku),
	- za pomocą odpowiednich narzędzi wydających alerty (auditd, AIDE, Wazuh, OSSEC, EDR).

## Analiza wpisów w pliku `/etc/shadow` (Case Study)

Fragment `/etc/shadow`:  
	admin:$6$KlmNopQ...:17000:0:90:7:::  
	backup:$6$UvWxYz12...:19600:0:99999:7:::  
	www-data:*:19500:0:99999:7:::  
	contractor:$6$GhJLeI36G38...:19500:10:30:19690::  

Analiza:  
- admin - brak realnej rotacji hasła (lastchg sprzed lat),
	- możliwe uwierzytelnianie poza hasłem (np. SSH key),
	- potencjalne 'stale account',
	- wymaga weryfikacji aktywności (logi, SSH, sudo),
- backup - konto systemowe z hashem hasła (nietypowe) -> wymaga weryfikacji przeznaczenia,
- contractor - nietypowa wartość pola inactive (19690) -> możliwa błędna konfiguracja lub nieprawidłowa interpretacja pola,
- www-data - poprawne ustawienia.   

Sugerowane działania:   
- konto admin - wskazany przegląd pod kątem zasadności jego dalszego utrzymania. Jeśli nadal jest wymagane, lecz nie ma potrzeby logowania za pomocą hasła (np. wykorzystywane wyłącznie z kluczem SSH lub przez procesy systemowe), należy rozważyć zablokowanie możliwości logowania hasłem `passwd -l`, co zmniejszy powierzchnię ataku,
- konto backup - ustalić, czy konto jest przeznaczone do logowania interaktywnego. Jeśli nie, można rozważyć blokadę hasła, 
- konto contractor - dostosować wartość inactive do realnych potrzeb (np. kilkadziesiąt dni), aby konto nie pozostawało aktywne przez nieproporcjonalnie długi okres po wygaśnięciu hasła. 