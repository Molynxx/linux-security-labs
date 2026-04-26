# /etc/security/faillock.conf

## Cel laboratorium
Celem laboratorium jest zapoznanie się z plikiem konfiguracyjnym modułu `pam_faillock.so` oraz jego rolą w systemie.

## Czym jest `pam_faillock.com`
jest to moduł zabezpieczający -  wymusza politykę bezpieczeństwa (np. blokada po 5 nieudanych próbach logowania) i generuje zdarzenia, które są monitorowane w SIEM. Moduł prowadzi pliki z licznikiem dla każdego użytkownika w `/var/run/faillock/`. Gdy ktoś wpisuje złe hasło licznik się iteruje, a gdy zostanie osiągnięty limit (deny) konto jest blokowane na czas określony w opcji `unlock_time`. 
Moduł `pam_faillock.so` posiada różne tryby modułu:
- `preauth` - w stosie przed zapytaniem o hasło, moduł sprawdza plik licznika i jeśli został przekroczony natychmiast przerywa logowanie nie pytając o hasło,
- `authfail` - w stosie po nieudanym sprawdzeniu hasła przez `pam_unix.so`, zwiększa licznik nieudanych prób dla tego użytkownika, 
- `authsucc` - w stosie po udanym sprawdzeniu hasła przez `pam_unix.so`, zeruje licznik nieudanych prób dla tego użytkownika. 

## Plik konfiguracyjny modułu pam_faillock.so
W nowoczesnych systemach opcja dla modułu znajduje się w pliku `/etc/security/faillock.conf`. W pliku można określić wartości opcji zabezpieczeń: 
- `deny` - liczba nieudanych prób po których konto jest zablokowane (np. deny=5),
- `unlock_time` - to czas blokady w sekundach, po tym czasie próby są resetowane (np. unlock_time=600), konto odblokuje się po 10 minutach,
- `fail_interval` - okres (w sekundach), w którym zliczane są próby. Domyślnie 900 sekund (np. fail_interval=900),
- `even_deny_root` - bardzo ważne, domyślnie root nie jest blokowany, aby uniknąć DoS, ta opcja włącza blokadę dla konta root, w systemach wysokiego ryzyka powinna być włączona (np. even_deny_root,
- `silent` - nie wyświetla użytkownikowi komunikaty o blokadzie konta (np. silent), 
- `dir` - ścieżka do katalogu z plikami liczników (np. /var/run/faillock), przydatne przy diagnostyce.  
Przykład poprawnie skonfigurowanego pliku faillock.conf:  
	deny=5  
	unlock_time=600  
	fail_interval=900  
	even_deny_root  

## Gdzie szukać logów
Logi znajdują się w pliku `/var/log/auth.log`, przykładowe logi:
- próby dla użytkownika `jan` (który nie istnieje):  
	sshd[12345]: pam_faillock(sshd:auth): User unknown
- próby dla użytkownika `jan` (istnieje):  
	sshd[12345]: pam_faillock(sshd:auth): authentication failure; user=jan
- po przekroczeniu limitu:  
	sshd[12345]: pam_faillock(sshd:auth): User jan has been locked for 600 seconds

## Wnioski bezpieczeństwa
- kluczowa opcja `even_deny_root` - zawsze należy sprawdzać czy jest włączona, to wymóg wielu polityk bezpieczeństwa, bez niej root jest chroniony tylko siłą hasła, 
- należy przestrzegać zgodności limitów, wartości `deny` i `unlock_time` muszą być zgodne z polityką firmy,
- dobrą praktyką są ustawienia pliku `faillock.conf` zamiast pisać długie linie w `common-auth`, to ułatwia audyt,
- logi blokad i prób znajdują się w `/var/log/auth.log`, wpisy z `pam_faillock`. 

## Case study - pam_faillock w środowisku produkcyjnym
### Kontekst:  
Firma ma serwer produkcyjny (Ubuntu 22.04) z dostępem SSH dla 3 adminów: anna, jan i admin.   
- Polityka bezpieczeństwa wymaga:
	- blokada konta po 5 nieudanych próbach logowania, 
	- blokada trwa 10 minut (600 sekund)
	- okno zliczenia prób - 15 minut (900 sekund)
	- blokada dotyczy również konta root 
- Stan faktyczny:
	- plik `/etc/security/faillock.conf`:  
		deny=6  
		unlock_time=300   
		fail_interval=1800  
		'#' even_deny_root  
	- plik /etc/pam.d/common-auth:  
		auth required pam_faillock.so preauth  
		auth [success=1 default=ingore] pam_unix.so  
		auth required pam_faillock.so authfail 
		auth sufficient pam_faillock.so authsucc  
	- logi z `/var/log/auth.log`: 
	  Mar 26 02:00:01 host sshd[12345]: pam_unix(sshd:auth) authentication failure; user=anna  
	  Mar 26 02:00:05 host sshd[12346]: pam_unix(sshd:auth) authentication failure; user=anna  
	  Mar 26 02:00:09 host sshd[12347]: pam_unix(sshd:auth) authentication failure; user=anna  
	  Mar 26 02:00:13 host sshd[12348]: pam_unix(sshd:auth) authentication failure; user=anna  
	  Mar 26 02:00:17 host sshd[12349]: pam_unix(sshd:auth) authentication failure; user=anna  
	  Mar 26 02:00:21 host sshd[12350]: pam_unix(sshd:auth) authentication failure; user=anna  
	  Mar 26 02:00:25 host sshd[12351]: pam_faillock(sshd:auth) user anna has been locked for 300 seconds  

	  Mar 26 02:05:01 host sshd[12400]: pam_unix(sshd:auth) authentication failure; user=root  
	  Mar 26 02:05:05 host sshd[12401]: pam_unix(sshd:auth) authentication failure; user=root  
	  Mar 26 02:05:09 host sshd[12402]: pam_unix(sshd:auth) authentication failure; user=root  
	  Mar 26 02:05:13 host sshd[12403]: pam_unix(sshd:auth) authentication failure; user=root  
	  Mar 26 02:05:17 host sshd[12404]: pam_unix(sshd:auth) authentication failure; user=root  
	  Mar 26 02:05:21 host sshd[12405]: pam_unix(sshd:auth) authentication failure; user=root  
	 

	  Mar 26 02:10:01 host sshd[12450]: pam_unix(sshd:auth) authentication failure; user=admin  
	  Mar 26 02:10:05 host sshd[12451]: pam_unix(sshd:auth) authentication failure; user=admin  
	  Mar 26 02:10:09 host sshd[12452]: pam_unix(sshd:auth) authentication failure; user=admin  
	  Mar 26 02:10:13 host sshd[12453]: pam_unix(sshd:auth) authentication failure; user=admin  
	  Mar 26 02:10:17 host sshd[12454]: pam_unix(sshd:auth) authentication failure; user=admin  
	  Mar 26 02:10:21 host sshd[12455]: pam_unix(sshd:auth) authentication failure; user=admin  
	- Dodatkowe informacje z wywiadu:
		- serwer ma otwarty SSH na porcie 22 (domyślnie)
		- brak firewall blokującego podejrzane IP
		- anna zgłosiła, że nie mogła się zalogować w nocy, ale rano już działało
		- root i admin nie zgłaszali problemów
		- w logach widać, że wszystkie nieudane próby pochodziły z tego samego adresu IP (203.0.113.45)  

Analiza:  
- konfiguracja nie spełnia warunków polityki bezpieczeństwa, ponieważ:
	- deny jest ustawione na 6 (nie spełnia wymogu blokowania po 5 nieudanych próbach) -> należy zmienić wartość deny na 5,
	- blokada konta wynosi 300 sekund (5 minut), polityka wymaga 600 sekund -> należy zmienić wartość na 600,
	- okno zliczania prób jest ustawione na 1800 sekund (30 minut), a polityka wymaga 900 sekund -> należy zmienić na 900 sekund,
	- najniebezpieczniejszy błąd to zakomentowana  linia z opcją even_deny_root, to oznacza bowiem ze opcja jest wyłączona -> należy usunąć '#' by włączyć opcję. 
- Analiza logów:
	- prób było 6, ponieważ ustawiono taką wartość deny w pliku konfiguracyjnym modułu,
	- konto anna zostało zablokowane na 300 sekund (5 minut) po przeprowadzeniu 6 nieudanych prób, 
	- konto root nie zostało zablokowane ponieważ opcja `even_deny_root` w pliku konfiguracyjnym była wyłączona, ponadto atakujący po próbie na koncie anna nie przeprowadził 7 próby,
	- konto admin nie zostało zablokowane, ponieważ atakujący po próbie brute force na koncie anna wiedział już jaki jest limit deny, nie próbował 7 raz wpisać danych logowania,
	- z logów wynika, że żadne konto nie zostało skompromitowane, nie widać otwarcia sesji, choć może to wynikać z braku modułu `pam_systemd.so` w typie session. Należy więc sprawdzić konfigurację PAM typu session, (pliki common-session oraz sshd) czy któryś zawiera moduł 'pam_systemd.so', jeśli tak konta nie zostały skompromitowane, jeśli nie należy zmienić hasła do tych kont, zweryfikować działania przeprowadzone a obu tych kontach od momentu ataku brute force i wycofać ewentualne nieautoryzowane zmiany w systemie,
	- należy zmienić wartości w pliku konfiguracyjnym faillock.con by dostosować wpisy do wymogów polityki bezpieczeństwa.
- dodatkowo warto zaznaczyć, że port dla SSH jest ustawiony na domyślny, to port często skanowany, należy rozważyć zmianę portu na inny. 

