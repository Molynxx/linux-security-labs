# /etc/security/pwquality.conf

## Cel laboratorium 
Celem tego laboratorium jest zapoznanie się z działaniem modułu `pam_pwquality.so` oraz jego konfiguracją. 

## Czym jest moduł pam_pwquality.so 
Jest to moduł PAM wymuszający złożoność ustawianych haseł, moduł ten uruchamia się przy zmianie hasła (passwd). Moduł znajduje się w pliku common-password, zwykle przed modułem `pam_unix.so`.   
Przykład zastosowania modułu w pliku common-password:    
	requisite pam_pwquality.so retry=3  
	required pam_pwhistory.so remember=5  
	required pam_unix.so use_authtok  
Opcja `authtok` dla modułu `pam_unix.so` zapewnia, że moduł pracuje na haśle z poprzedniego modułu, nie pyta ponownie o hasło.  

## Czym jest plik /etc/security/pwquality.conf
To plik konfiguracyjny modułu `pam_pwquality.so`. W pliku konfiguruje się opcje dla modułu, celem zagwarantowania złożoności ustawianego hasła. Wśród opcji wyróżnić można:
- `minlen` - minimalna długość hasła,
- `minclass` - minimalna liczba klas znaków, czyli małe, duże, cyfry, znaki specjalne, 
- `difok` - liczba znaków, które muszą się różnić od starego hasła,
- `maxrepeat` - maksymalna liczba powtarzających się znaków,
- `maxclassrepeat` - maksymalna liczba powtórzeń jednego znaku w haśle,
- `usercheck` - hasło nie może zawierać nazwy użytkownika,
- `enforcing` - czy wymuszać politykę - 1=tak, 0=tylko ostrzeżenie,
- `credit` - wymagana ilość cyfr w haśle,
- `ucredit` - wymagana ilość wielkich liter w haśle,
- `lcredit` - wymagana ilość małych liter w haśle,
- `ocredit` - wymagana ilość znaków specjalnych w haśle,
- `gecocheck` - sprawdza imię, login i dane z `/etc/passwd` i nie pozwala używać ich w haśle, 
- `dictcheck` - sprawdza hasło wg słownika, chroni na przykład przed hasłami słownikowymi np. 'Password123',
- `retry` - ile razy użytkownik może spróbować wprowadzić nowe hasło, 
- `enforce_for_root` - domyślnie root nie podlega polityce jakości haseł, dzięki tej opcji można włączyć te polityki dla roota,
Zaleca się by ustawienia konfigurować w pliku konfiguracyjnym a nie ujmować w opcjach moduły w plikach PAM.   

## Przykładowa konfiguracja pliku /etc/security/pwquality.conf
- poprawna przykładowa konfiguracja pliku:
	minlen=12  
	minclass=2  
	difok=3  
	maxrepeat=2  
	usercheck=1  
	enforcing=1  

## Logi
Wszelkie błędy są zwracane do `/var/log/auth.log`. Przykładowe wpisy:  
passwd: pam_pwquality(passwd:chauthtok): password fails complexity check: too short  
passwd: pam_pwquality(passwd:chauthtok): password fails complexity check: not enough character class  

## Wnioski bezpieczeństwa
- opcje modułu `pam_pwquality.so` są bardzo istotne dla bezpieczeństwa dlatego należy weryfikować ich poprawne ustawienie (zwłaszcza po incydentach, atakujący mógł zmienić opcje jeśli uzyskał dostęp do konta z `sudo`),
- należy weryfikować czy moduł jest użyty w pliku common-password i czy jest użyty w odpowiedniej kolejności (umieszczenie za unix, może spowodować ze opcje ustawione dla modułu nie zadziałają),
- z perspektywy bezpieczeństwa przed atakami brute force, najważniejszym parametrem jest `minlen` (długość hasła). Złożoność (`minclass`) pomaga głównie przeciwko atakom słownikowym i zgadywaniu, ale nie zastąpi długiego hasła. W systemach wysokiego ryzyka zaleca się `minlen=16` niezależnie od `minclass`.

## Case study

Polityka wymaga: minlen=12, minclass=3, difok=3,  
Konfiguracja: minlen=8, minclass=2, difok=2, brak usercheck.  
Użytkownik zmienia hasło z Anna2024! na Anna2025!

Analiza:  
- wymogi polityki nie są spełnione, konfiguracja różni się od wymogów, 
- użytkownik, przy obecnej konfiguracji będzie mógł zmienić hasło na Anna2025!, ponieważ konfiguracja nie używa usercheck ani gecocheck. Ponadto długość hasła jest ustawiona na 8 (co jest spełnione przez nowe hasło), oraz pozwala na zmianę tylko dwóch znaków w stosunku do poprzedniego hasła, 
- konfiguracja nie jest bezpieczna i pozwala na ustawianie słabych haseł (krótkie, mało klas znaków, podobne do starych),
- zalecenia, dostosować opcje do wymogów polityki bezpieczeństwa, rozważyć dodanie opcji usercheck lub gecocheck a także dictchceck. Warto również rozważyć opcję ustawienia limitu prób wpisania nowego hasła (retry) oraz opcji enforcing celem zwiększenia bezpieczeństwa.
- w opisanym przypadku nie ma ustawionej opcji wymuszania polityki dla root, warto to rozważyć. 