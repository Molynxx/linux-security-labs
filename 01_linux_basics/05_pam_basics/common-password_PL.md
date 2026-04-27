# /etc/pam.d/common-password

## Cel laboratorium
Celem tego laboratorium było zaznajomienie się z rolą jaką pełni typ password w systemie oraz jakie występują w nim moduły i jakie zagrożenia są z nim związane. 

## Czym jest `/etc/pam.d/common-password`
Typ password odpowiada za zmianę, zapis i weryfikację jakości haseł użytkowników. Uruchamia się tylko przy tworzeniu lub zmianie hasła, a jego poprawna konfiguracja jest kluczowa dla bezpieczeństwa systemu. 

## Moduły typu password i ich kolejność
- `pam_pwquality.so` - odpowiada za jakość hasła,
- `pam_pwhistory.so` - zapobiega ponownemu użyciu poprzednich haseł, 
- `pam_sss.so` - zapisuje nowe hasła w zewnętrznej bazie (dotyczy konta domenowego),
- `pam_unix.so` - zapisuje nowe hasło w `/etc/shadow` w postaci hasha, 
- `pam_gnome_keyring.so` - synchronizuje hasło systemowe z keyringiem użytkownika, nie wpływa na bezpieczeństwo systemowe (wyłącznie dla kont lokalnych).

## Flagi kontrolne modułów
Moduły `pam_pwhistory.so` oraz `pam_unix.so` powinny mieć flagę `required`. Moduł `pam_sss.so` w niektórych przypadkach może występować z flagą `sufficient`, moduł `pam_pwquality.so` zazwyczaj posiada flagę `requisite` (hasło niespełniające polityk bezpieczeństwa musi być odrzucone), natomiast moduł `pam_gnome_keyring.so` występuje zwykle z flagą `optional`, ponieważ moduł ten nie ma wpływu na bezpieczeństwo. 

## Zagrożenia common-password
Konfiguracja modułów jest bardzo istotna, w przypadku złej konfiguracji mogą wystąpić zagrożenia:
- zbyt słabe hasło ustawione przez użytkownika, 
- powtarzanie używanego już wcześniej hasła, 
- w przypadku braku modułu `pam_sss.so` - brak zapisu hasła w zewnętrznej bazie (nieskuteczna zmiana hasła),
- słaby hash w `/etc/shadow` lub brak zapisu hasła w przypadku braku modułu `pam_unix.so`,
- błędne flagi kontrolne modułów - omijanie istotnych polityk bezpieczeństwa, 
- brak `use_authtok` - moduły mogą otrzymywać różne wersje hasła.

## Wnioski bezpieczeństwa
- sprawdzenie, czy wszystkie krytyczne moduły znajdują się w stosie i mają poprawne flagi, 
- weryfikacja, czy moduły używają opcji `use_authok`. UWAGA: opcja `use_authtok` dotyczy modułów `pam_unix.so`, `pam_pwhistory.so`, `pam_sss.so`, lecz nie dotyczy modułu `pam_pwquality.so`,  
- umieszczenie wszystkich istotnych modułów w pliku `common` to dobra praktyka, 
- regularne sprawdzanie konfiguracji pod kątem bezpieczeństwa hasła i jego zapisu. 

## Przykład analizy konfiguracji pliku common-password (Case Study)

password sufficient pam_unix.so   
password required pam_pwquality.so    
password required pam_pwhistory.so use_authtok    
password sufficient pam_sss.so use_authtok    

Analiza:    
Skrajny błąd - pierwszy moduł `pam_unix.so` z flagą sufficient pozwala na zapisanie dowolnego hasła, pomijając polityki jakości i historii haseł, co może prowadzić do słabych haseł w systemie i utrudnia audyt SOC/IR.   

Sugerowane działanie:    
- zmienić flagę modułu pam_unix.so na 'requisite' lub 'required', 
- przenieść moduł pam_unix.so na koniec stosu, 
- dodać do modułu pam_unix.so opcję use_authtok.

