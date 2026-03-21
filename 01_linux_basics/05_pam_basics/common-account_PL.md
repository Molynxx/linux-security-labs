# /etc/pam.d/common-account

## Cel
Celem laboratorium jest zrozumienie czym jest plik `/etc/pam.d/common-account` oraz zapisów znajdujących się w nim.

## Czym jest /etc/pam.d/common-account
Common-account decyduje czy konto może zostać użyte, nawet jeśli uwierzytelnianie przebiegło pomyślnie. Jest to warstwa polityk i kontroli, to ustawienia globalne mogące być załączone do plików konfiguracji PAM dla aplikacji. Występuje on zawsze po fazie uwierzytelniania `auth`.

## Moduły i ich kolejność w pliku common-account 
Wśród modułów występujących w tym pliku wyróżnić można:
- `pam_nologin.so` - blokuje dostęp przy `/etc/nologin`,
- `pam_faillock.so` - blokuje po nieudanych próbach logowania, 
- `pam_unix.so` - sprawdza status konta lokalnego,
- `pam_sss.so` - sprawdza status konta zdalnego (AD/LDAP),
- `pam_deny.so` - wymusza odrzucenie dostępu,
- `pam_permit.so` - jawne przyznanie dostępu po spełnieniu wszystkich warunków. 
	

## Flagi kontrolne w pliku common-account
Podstawowa flaga w common-account to `required` - wszystko musi się udać, by dostęp został przyznany. Wyjątki: moduły, które powinny przerwać dostęp natychmiast (np. `pam_nologin.so`, `pam_deny.so`) posiadają flagę `requisite`. 

## Zagrożenia związane z plikiem common-account
- `pam_nologin.so` - uniemożliwia zwykłym użytkownikom dostęp do systemu w momencie, kiedy system może być wrażliwy (np. jest w trybie `maintenance`). Odmowa dostępu nie dotyczy użytkownika root,
- brak modułu `pam_faillock.so` - brak blokady po nieudanych logowaniach (Red Flag),
- pam_unix.so i `pam_sss.so` - brak tego modułu -> brak sprawdzenia statusu konta, ryzyko potencjalnego nieautoryzowanego dostępu, 
- niewłaściwe flagi - pomijanie istotnych modułów. 

## Wnioski
- Moduł `pam_nologin.so` powinien być na początku typu, by od razu blokować dostęp w trybie `maintenance`,
- moduł `pam_faillock.so`  i pam_unix.so muszą występować dla poprawnej walidacji kont lokalnych,
- moduł `pam_sss.so` musi występować dla poprawnej walidacji kont zdalnych, 
- moduł `pam_deny.so` jest wymagany dla właściwego zabezpieczenia przed przyznaniem dostępu dla kont nieautoryzowanych, 
- brak modułu `pam_permit.so` to ryzyko niepoprawnego działania systemu, 
- flagi modułów powinny być poprawnie ustawione (`required`, `requisite`), aby unikać pominięcia istotnych kontroli. 

## Analiza konfiguracji pliku common-account (Case Study)

account required pam_faillock.so    
account sufficient pam_unix.so    
account required pam_sss.so    
account requisite pam_deny.so    

Analiza:   
Nie wszystkie konsekwencje tego zapisu są oczywiste. To, co oczywiste to:
- brak modułu `pam_nologin.so` stanowi zagrożenie dla systemu gdy jest w trybie maintenance,
- moduł `pam_faillock.so` jest ustawiony we właściwym miejscu i mam poprawnie ustawioną flagę,
- flaga modułów `pam_sss.so` i `pam_deny.so` są ustawione poprawnie, podobnie jak kolejność tych modułów w zapisie. 
- flaga `sufficient` dla modułu `pam_unix.so` jest nieprawidłowa. 
Co jest nieoczywiste:
Plik globalny common-account zwykle jest jedynie częścią konfiguracji PAM dołączaną do plików konfiguracji PAM dla aplikacji. `Sufficient` w module `pam_unix.so` powoduje pominięcie całego stosu  modułów w typie account (nie tylko modułów w pliki common). Konsekwencje takiego zapisu:
- pominięcie modułów za `pam_unix.so`, gdyby nie było więcej zapisów w pliku PAM aplikacji nie stwarza zagrożenia,
- gdyby jednak poza plikiem common było ich więcej, to konto lokalne nie było by już w żaden więcej sposób sprawdzane typem account,
- lokalni użytkownicy nie muszą przechodzić przez całą drogę walidacji jak użytkownicy domenowi,
- dodane w przyszłości moduły nie będą dotyczyć kont lokalnych.  

Sugerowane działania:
- zmienić flagę dla modułu `pam_unix.so` na `required`.
