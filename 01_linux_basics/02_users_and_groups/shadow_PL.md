## /etc/shadow

## Cel laboratorium
Celem tego laboratorium było zapoznanie się z wpisami w pliku `/etc/shadow`, zrozumienie znaczenia wpisów, oraz praktyka ich analizy.

## Czym jest /etc/shadow
Plik `/etc/shadow` zawiera informacje o:
- blokadach kont,
- danych dotyczących zmiany hasła, 
- wymuszeniu zmiany hasła,
- wygaśnięciu konta.  
  
Szczegółowe informacje w każdej linii `/etc/shadow` (oddzielone dwukropkiem `:` ):
- login użytkownika,
- hash hasła, żeby zrozumieć jaki algorytm hashujący ma hasło, należy sprawdzić jakie symbole występują na początku:
	- $1$ -> MD5 przestarzały (czerwona flaga),
	- $5$ -> SHA-256 crypt oparty na SHA-256, z 5000 iteracji - lepszy niż goły SHA256, ale wciąż za szybki jak na dzisiejsze standardy, przestarzały, niezalecany - warto migrować na yescrypt/bcrypt,
	- $6$ -> SHA-512 crypt oparty na SHA-512, domyślnie 5000 iteracji - lepszy niż goły SHA-512, ale podatny na GPU brute force, akceptowalny, ale nie optymalny - w nowych wdrożeniach zaleca się yescrypt lub bcrypt,
	- $2b$ -> bcrypt dobry, odporny na brute force,
	- $y$ -> yescrypt (glibc >= 2.35) nowoczesny, obecnie zalecany w nowoczesnych systemach Linux. 
- lastchg - ostatnia zmiana hasła (dni od 01.01.1970 roku),
- min - minimalna liczba dni między zmianami hasła,
- max - maksymalna liczba dni ważności hasła,(jeśli użytkownik nie zmieni hasła przed terminem jego wygaśnięcia to po tym terminie podczas logowania system wymusi zmianę hasła),
- warn - ostrzeżenie o wygaśnięciu hasła - ile dni przed wygaśnięciem hasła system informuje użytkownika o konieczności zmiany hasła,
- inactive - nieaktywność  - ile dni po wygaśnięciu hasła konto zostaje zablokowane, 
- reserved - ostatnie pole - rzadko używane, zwykle puste.

## Wnioski bezpieczeństwa
- Na co zwrócić uwagę w pliku `/etc/shadow`:
	- czy konto systemowe lub interaktywny użytkownik ma hasło czy jest zablokowane ( `!`,`!!`,`!*`), 
	- czy konto interaktywne ma wymuszoną zmianę hasła i co jaki czas,
	- czy ustawione są ostrzeżenia przed wygaśnięciem hasła,
	- czy jest ustawiona ilość dni do zablokowania konta po wygaśnięciu hasła,
	- kiedy ostatni raz było zmienione hasło,
	- jaki jest użyty algorytm hashujący,
- Brak hasła lub ustawienie zbyt długiego maksymalnego okresu hasła (max days)może stanowić ryzyko bezpieczeństwa,
- Aby sprawdzić szczegóły konta pod kątem ważności hasła, można użyć polecenia: `chage -l nazwa_użytkownika`.
- słaby algorytm hashujący może być podatny na brute force, najlepszym wyborem jest tutaj yescrypt i bcrypt. To wolne algorytmy, które są specjalnie zaprojektowane do przechowywanie haseł i są odporne na ataki brute force offline dzięki swojej kosztowności obliczeniowej. Algorytm MD5 zawsze powinien zaalarmować SOC/IR.

## Potencjalne zagrożenia
- Konto systemowe z aktywnym hashem zamiast blokady (`!`, `!!`, `*`) - weryfikacji przeznaczenia i uprawnień. Należy wtedy sprawdzić w pliku `/etc/passwd` czy konto ma dostęp do `/bin/bash`. To jednak nie jest równoznaczne z atakiem, zdarza się, że może to być: 
	- konto serwisowe,
	- konto aplikacyjne,
	- migracja systemu,
	- konfiguracja legacy (odziedziczone / stare / historyczne rozwiązanie).
	IR zawsze bada kontekst:
	- kto stworzył konto,
	- kiedy,
	- czy jest w sudo,
	- czy logowało się przez SSH,
	- czy ma klucz w `~/.ssh`. 
- Konta systemowe z czasową blokadą logowania (`!`) mogą zostać odblokowane przez osobę nieuprawnioną jeśli atakujący uzyskał uprawnienia administracyjne, tutaj również należy sprawdzić w `/etc/passwd` czy konto ma dostęp do `/bin/bash`. Jeśli tak to potencjalne zagrożenie. 
- Konta interaktywnych użytkowników, które nie mają wymuszonej zmiany hasła lub nie mają go wcale i mają bardzo długie maksymalne dni ważności również stanowią potencjalne zagrożenie. 

## Przykłady zabezpieczeń dla kont podejrzanych 
- Zablokowanie interaktywnego shella: `sudo usermod -s /usr/sbin/nologin nazwa_użytkownika`,
- Trwałe zablokowanie hasła: `sudo passwd -l nazwa_użytkownika`,
- Ograniczenie dostępu do SSH - w pliku `/etc/ssh/sshd_config ustawić`: `DenyUsers nazwa_użytkownika`,
- Sprawdzenie ustawień sudo: `sudo -l -U nazwa_użytkownika`,
- monitorowanie zmian w shadow:
	- ręcznie: `stat /etc/shadow` zwracamy tutaj uwagę na godzinę zmian, zmianę właściciela, zmianę zawartości pliku, zmiany uprawnień pliku,
	- za pomocą odpowiednich narzędzi wydających alerty (np. auditd, AIDE, Wazuh, OSSEC, EDR).

## Analiza wpisów w pliku /etc/shadow (case study)

### Case study 1
```
Fragment /etc/shadow  
root:!:19239:0:99999:7:::  
alice:$6$abcd123$zYx...:19345:0:90:7:::  
syslog:*:19200:0:99999:7:::  
   ```
Analiza:   
- root konto uprzywilejowane, hasło zablokowane (`!`), logowanie hasłem niemożliwe. Jest to często spotykana praktyka w systemach produkcyjnych, gdzie preferuje się dostęp przez sudo lub klasyczne klucze SSH. Konfiguracja wygląda poprawnie.
- alice - użytkownik interaktywny, hasło hashowane algorytmem SHA-512, maksymalna ważność hasła 90 dni, ostrzeżenie 7 dni przed wygaśnięciem, Konfiguracja zgodna z praktyką. 
- syslog - konto systemowe z trwałą blokadą hasła (`*`), brak możliwości logowanie hasłem. Ustawienie prawidłowe.  
  
Sugerowane działania:  
- brak konieczności podejmowania działań. 
- opcjonalnie w środowiskach wymagających wyższego poziomu bezpieczeństwa można rozważyć stosowanie wolniejszych algorytmów hashujących (np. yescrypt) oraz dostosować maksymalny okres ważności hasła zgodnie z polityką organizacji. 

### Przypadek 2  
```
Fragment /etc/shadow:  
bob:$6$abcd1234$E4Jd...KfD:18500:7:120:14:::  
backup:*:18450:0:99999:7:::  
daemon:!:18200:0:99999:7:::  
  ```
Analiza:    
- bob - użytkownik interaktywny, algorytm SHA-512, ważność hasła 120 dni, jego zasadność zależy od przyjętej polityki bezpieczeństwa. 
- backup - użytkownik systemowy, trwała blokada logowania za pomocą hasła (standardowa praktyka), pozostałe dane poprawne,
- daemon - usługa systemowa, zablokowane hasło, nie można się zalogować przy jego użyciu, konfiguracja dopuszczalna.   
   
Sugerowane działania:  
- dla konta bob można dostosować maksymalny czas ważności hasła do wewnętrznej polityki (np 90 dni), jeśli organizacja tego wymaga.
- dla użytkownika daemon warto zweryfikować, czy:
	- nie posiada powłoki interaktywnej,
	- nie ma nieuzasadnionych uprawnień (np. sudo). 

### Przykład 3 
```
Fragment /etc/shadow:  
david:$6$Zx91kLmN$F3nQ2pXz1...:19850:0:99999:7:::  
service1:!:19800:0:99999:7:::  
postgres:*:19620:0:99999:7:::   
intern:$6$AbCdEfGh$kl9mN8pQ...:19870:0:30:5:19950::  
```
Analiza:  
- dawid - konto interaktywne, algorytm hashujący SHA-512, zdecydowanie zbyt długi okres ważności hasła (99999), brak ustawień inactive, więc konto nie zostanie zablokowane po wygaśnięciu hasła. 
- service1 - konto systemowe z zablokowanym hasłem `!`, logowanie hasłem niemożliwe, ustawienie dopuszczalne,
- postgres - konto systemowe z trwałą blokadą hasła (`*`), ustawienia są poprawne,
- intern - konto interaktywne, algorytm hashowania SHA-512, bardzo długi czas pomiędzy wygaśnięciem hasła a blokadą konta.   
  
Sugerowane działania:  
- dla konta david wskazane jest dostosowanie maksymalnego okresu ważności hasła do wewnętrznej polityki (np. 60 lub 90 dni),
- dla konta service warto zweryfikować:
	- czy nie posiada interaktywnej powłoki,
	- czy nie posiada nieuzasadnionych uprawnień (np. `sudo`),
- dla konta intern rozważyć dostosowanie blokady konta po wygaśnięciu hasła do wewnętrznej polityki. 

### Przypadek 4
```
Fragment /etc/shadow:
admin:$6$KlmNopQ...:17000:0:90:7:::  
backup:$6$UvWxYz12...:19600:0:99999:7:::  
www-data:*:19500:0:99999:7:::  
contractor:$6$GhJLeI36Ge8...:1950:10:30:7:19690::  
```
Analiza:   
- konto admin ma ustawioną ważność na 90 dni, ostatnia zmiana hasła na tym koncie była kilka lat temu. Konto nie zostało zablokowane, ponieważ pola inactive i expire nie są ustawione. Ostatnia zmiana hasła miała miejsce wiele lat temu, co oznacza brak skutecznej zmiany hasła w tym okresie. Warto jednak zweryfikować aktywność konta, ponieważ istnieją mechanizmy umożliwiające korzystanie z konta bez logowania hasłem (np. uwierzytelnianie kluczem SSH). Ponadto konto może być wykorzystywane przez zadania cron lub inne procesy systemowe. W przypadku braku uzasadnienia biznesowego dla utrzymania konta, należy rozważyć jego dezaktywację lub usunięcie zgodnie z polityką organizacji.
- konto systemowe backup nie powinno mieć hasha hasła. W przypadku kont systemowych stosuje się blokadę hasła (`*` lub `!`), dlatego warto zweryfikować przeznaczenie tego konta.
- konto interaktywne contractor ma w polu lastchg ustawione 1950 - to podejrzanie niska wartość. Oznacza, że ostatnia zmiana hasła wskazuje na rok 1975, co jest niemożliwe - wpis jest najpewniej spreparowany lub to błąd w danych (czerwona flaga). Dodatkowo pole expire = 19690 wskazuje na datę wygaśnięcia konta ok. 2023 roku - konto wygasł. Należy to skorelować z `/etc/passwd`,
- konto www-data ma poprawne ustawienia.  
   
Sugerowane działania:  
- dla konta admin wskazany jest przeglądu pod kątem zasadności jego dalszego utrzymania. jeśli konto jest nadal wymagane, lecz nie ma potrzeby logowania za pomocą hasła (np. wykorzystywane wyłącznie z kluczem SSH lub przez procesy systemowe), należy rozważyć zablokowanie możliwości logowania hasłem (`passwd -l`), co zmniejszy powierzchnię ataku. 
- dla konta backup, należy ustalić, czy konto jest przeznaczone do logowania interaktywnego. Jeśli nie można rozważyć blokadę hasła. 
- dla konta contractor należy zweryfikować integralność danych w `/etc/shadow` - lastchg=1950 jest niemożliwe, a expire=19690 wskazuje, że konto już wygasło. Należy skorelować z `/etc/passwd` i sprawdzić, czy konto jest nadal aktywne.   