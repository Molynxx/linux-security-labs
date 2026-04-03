# Flagi kontrolne

## Cel
Zrozumienie czym są flagi kontrolne, jakie są ich rodzaje oraz jaki mają wpływ na funkcjonowanie polityk PAM.

## Czym są flagi kontrolne
Flagi PAM definiują logikę decyzyjną, określając czy wynik działania modułu powoduje kontynuację, przerwanie lub zakończenie procesu uwierzytelniania. Flagi kontrolne dzielą się na:
- standardowe (`required`, `requisite`, `sufficient`, `optional`),
- rozszerzone (np. `[success=1 default=ignore]`).

## Standardowe flagi kontrolne
- `required` - moduł musi się wykonać, nie przerywa wykonania typu, decyzja o pozytywnym zakończeniu zapada na końcu typu, jeśli żaden moduł `required` nie zakończył się błędem. 
- `requisite` - natychmiast przerywa wykonanie typu w przypadku niepowodzenia modułu, a typ zakończy się niepowodzeniem. 
- `sufficient` - jeśli moduł zakończy się sukcesem i nie było wcześniejszych błędów `required`, PAM kończy typ natychmiast sukcesem. 
- `optional` - nie wpływa na decyzję końcową typu (chyba że jest jedynym modułem lub jeśli inne moduły nie zwróciły jednoznacznego wyniku).

## Rozszerzone flagi kontrolne
To są flagi pozwalające ujmować sytuacje warunkowe if/else. Struktura zapisu takiej flagi to:  
	`[typ] [wynik=akcja wynik=akcja ...] moduł`  
Struktura zapisu składa się z wyniku i akcji.
- podstawowe wartości wyniku:
	- success - moduł zakończył się sukcesem,
	- ignore - wynik modułu jest ignorowany,
	- module_unknown - PAM napotkał moduł, którego nie zna lub nie może załadować i traktuje go jako błąd,
	- auth_err - błąd uwierzytelniania,
	- user_unknown - użytkownik nie istnieje, 
	- default - wszystkie nieprzewidziane przypadki. 
- podstawowe akcje:
	- ignore - PAM ignoruje wynik modułu i przechodzi do następnego modułu w tym samym typie. Nie wpływa na decyzję końcową typu. 
	- ok - wynik modułu traktowany jest jako sukces. PAM kontynuuje sprawdzanie pozostałych modułów w typie. 
	- bad - wynik traktowany jako błąd i wpływa na końcową decyzję typu, ale przetwarzanie może być kontynuowane. 
	- die - kończy cały stos PAM dla tej operacji. 
	- done - typ traktuje moduł jako ostatni decydujący moduł. PAM nie sprawdza już kolejnych modułów w tym typie. 
	- 1, 2 (skip) - przeskakuje kolejne N modułów w tym typie. N jest liczbą - ile modułów ma być pominiętych. 

## Wnioski bezpieczeństwa 
- błędna flaga może pozwolić na:
	- brak możliwości logowania,
	- obejście uwierzytelnienia,
	- brak logowania krytycznych zdarzeń (audyt),
	- problemy z sesją lub przyznawaniem praw (`account`, `session`).
- by właściwie stosować flagi należy dobrze znać funkcje typów i modułów oraz ich kolejność, 
- należy znać dobrze wyniki i akcje rozszerzonych flag, żeby uniknąć błędów konfiguracji. 
Najbezpieczniejszą strategią konfiguracji PAM jest podejście default=deny, gdzie brak dopasowania skutkuje odmową dostępu. 

## Analiza przypadków (case study)

### Case study 1

auth required pam_unix.so  
auth requisite pam_faillock.so  
auth sufficient pam_ldap.so  
auth optional pam_permit.so  

Analiza:  
- analizowany typ - auth,
- obecna kolejność i sposób użycia `pam_faillock.so` nie gwarantują poprawnego zliczania nieudanych prób logowania dla wszystkich mechanizmów uwierzytelniania, ponieważ moduł nie jest użyty w pełnym schemacie (preauth / authfail / authsucc). 
- `pam_faillock.so` - powinien być użyty w odpowiednim schemacie (np. authfail), aby poprawnie zliczać próby logowania.
- `pam_permit.so` - moduł nie powinien być używany w systemach produkcyjnych w typie auth, ponieważ może prowadzić do obejścia uwierzytelnienia. 
- kolejność modułów ma wpływ na to, co zostanie policzone i zapisane w logach.   

Sugerowane działania:
- umieścić `pam_faillock.so` zgodnie z jego rolą (np. `preauth` przed uwierzytelnianiem, `authfail` po nieudanej próbie),
- dodać do modułu `pam_faillock.so` opcję `authfail` oraz zmienić jego flagę na `required`,
- usunąć z auth moduł `pam_permit.so` to może oznaczać potencjalny backdoor.
Po tych działaniach typ auth będzie się wykonywać w sposób poprawny i bezpieczny dla procesu uwierzytelniania. 

### Case study 2

auth [success=1 default=ignore] pam_unix.so  
account [success=ok user_unknown=die] pam_faillock.so  
auth [default=bad] pam_ldap.so   
session [ignore=ignore module_unknown=die] pam_limits.so   
  
Analiza:  
- kolejność typów jest nieprawidłowa, najpierw moduły typu `auth` a dopiero potem `account`,
- kolejność modułów `pam_unix.so` i `pam_ldap.so` zależy od polityki, 
- flaga [success=1 default=ignore] - poprawne w konfiguracjach hybrydowych (local + LDAP), jednak może utrudnić analizę błędów i logów.
- moduł `pam_faillock.so` - brakuje instrukcji co w przypadku błędu, potrzebne [default=bad], 
- moduł `pam_ldap.so` - poprawne ustawienie,
- moduł `pam_limits.so` - brakuje akcji dla success i default [success=ok default=bad],
- flagi rozszerzone powinny być stosowane tylko tam, gdzie potrzebna jest precyzyjna kontrola przepływu (np. pomijanie modułów lub konkretnych wyników). 
  
Sugerowane działania:
- zmienić kolejność typów na prawidłowy auth -> account -> session,
- w przypadku modułu `pam_unix.so` zależnie od polityki:
	- albo zostawić przed `pam_ldap.so` i pozostawić obecną flagę,
	- albo umieścić go za modułem `pam_ldap.so` i zmienić flagę na [success=ok default=bad],
- moduł `pam_faillock.so` - dodać do flagi wpis [default=bad],
- moduł `pam_limits.so` - zmienić flagę na [success=ok default=bad].