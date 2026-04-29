# /etc/security/access.conf

## Cel laboratorium
Zapoznanie się konfiguracją opcji dla modułu `pam_access.so`.

## Czym jest pam_access.so
Jest to moduł stosowany w typie `account` (common-account, sshd, gdm-password / lightdm, sudo), który odpowiada za kontrolę dostępu na podstawie:
- kto (użytkownik/grupa),
- skąd (host, IP, TTY, usługa),
- kiedy (ale to robi `pam_time.so` - osobny moduł). 

## Czym jest /etc/security/access.conf
Jest to plik konfiguracyjny modułu `pam_access.so`. Znajdują się w nim opcje zapisane w formacie:  
	<pozwolenie> : <kto> : <skąd>   
- pozwolenie: `+` (pozwól), `-` (odmów),
- kto:
	- nazwa użytkownika, 
	- @nazwa grupy, 
	- ALL (wszyscy),
	- LOCAL (użytkownicy lokalni),
	- EXCEPT (wyjątek, działa wewnątrz linii, nie na końcu)
- skąd:
	- ALL (wszędzie),
	- LOCAL (logowanie z konsoli / TTY bez sieci),
	- nazwa_hosta lub .domena (np .example.com),
	- IP lub CIDR (np. 192.168.1.0/24),
	- tty1, tty2... (konkretna konsola),
	- cron (dla zadań cron),
	- sudo (dla poleceń przez sudo).  
Uwaga: kolejność wpisów ma znaczenie - pierwszy pasujący decyduje. Na końcu domyślnie odmowa jeśli brak dopasowania. 

## Logi
Logi dla zdarzeń związane z tym modułem znajduję się w `/var/log/auth.log` (`/var/log/secure dla RHEL`). Przykładowe wpisy:  
	pam_access(sshd:account): access denied for user backuper from 203.0.113.45  
	pam_access(sshd:account): access denied for user root from :1  

## Wnioski bezpieczeństwa
- gdy moduł `pam_access.so` jest używany, należy weryfikować poprawność wpisów w `/etc/security/access.conf`, 
- zła kolejność linii może zablokować dostęp, no `- : ALL : ALL` przed `+ : admin :  ALL` może zablokować admina, 
- błąd składni wpisu w `access.conf` powoduje, że reguła jest ignorowana,
- EXCEPT w złym miejscu (działa wewnątrz pola nie na końcu reguły):
	- `- : ALL EXCEPT admin : ALL` - poprawny zapis,
	- `- : ALL : ALL EXCEPT admin` - zapis nieprawidłowy reguła jest ignorowana,

## Case study - konfiguracja dostępu za pomocą modułu pam_access.so oraz pliku access.conf
Kontekst:  
Firma ma serwer produkcyjny. Tylko dwóch inżynierów (jan_kowalski, anna_nowak) może się logować przez SSH z biura (sieć 10.10.10.0/24) lub z konsoli lokalnej. Żaden inny użytkownik nie może się logować przez SSH w ogóle. Root nie może się logować przez SSH. Wszyscy (w tym inżynierowie) mogą logować się przez konsolę lokalną.  

Zadanie:  
Napisać reguły w pliku `/etc/security/access.conf`, określić w którym pliku PAM umieścić moduł dla SSH, jak sprawdzić w logach czy reguły działają.  

Rozwiązanie:  
- Moduł `pam_access.so` należy umieścić w pliku PAM `sshd` w typie account z flagą `required`. 
- Reguły:  
	+ : jan_kowalski : 10.10.10.0/24  
	+ : anna_nowak : 10.10.10.0/24    
	+ : ALL : LOCAL  
	- : ALL : ALL
- sprawdzenie w logach czy reguły działają:  
	grep pam_access /var/log/auth.log  
	Przykładowy log gdy root próbuje SSH:  
	pam_access(sshd:account): access denied for user root from 10.00.10.5  
