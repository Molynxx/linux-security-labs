# /etc/pam.d/common-account

## Cel
Celem laboratorium jest zrozumienie czym jest plik `/etc/pam.d/common-account` oraz zapisów znajdujących się w nim.

## Czym jest /etc/pam.d/common-account
To nie jest plik sprawdzający hasła, to jest plik, który zaczyna egzekwować politykę PAM. `Common-account` decyduje czy konto może zostać użyte, nawet jeśli uwierzytelnianie przebiegło pomyślnie. Więc nawet gdy hasło jest poprawne lub klucz SSH jest prawidłowy to mimo tego dostęp może być odmówiony. `Common-account` to warstwa polityk i kontroli, to ustawienia globalne mogące być załączone do plików konfiguracji PAM dla aplikacji. Występuje on zawsze po fazie uwierzytelniania `auth`.

## Moduły i ich kolejność w pliku common-account 
Wśród modułów występujących w tym pliku wyróżnić można:
- `pam_nologin.so`:
	- sprawdza czy istnieje plik `/etc/nologin` (system w trybie maintenance),
	- jeśli istnieje plik nologin, odrzucony zostaje dostęp zwykłych użytkowników do systemu, niezależnie od poprawności uwierzytelnienia. 
- `pam_faillock.so` - to najważniejszy moduł obronny:
	- sprawdza czy konto jest zablokowane,
	- korzysta z danych zebranych w `auth`.
- `pam_unix.so` - sprawdza:
	- czy konto istnieje, 
	- czy nie jest zablokowane (`!`, `*` w pliku `/etc/shadow`),
	- czy nie wygasło.
	To nie jest uwierzytelnianie, to jest walidacja stanu konta. 
- `pam_sss.so` - dla kont domenowych:
	- czy konto jest aktywne w AD/LDAP,
	- czy nie zostało wyłączone, 
	- czy spełnia polityki domeny.
- `pam_deny.so`:
	- wymusza końcową decyzję `deny`, jeśli wcześniejsze moduły nie podjęły żadnej decyzji (np. nie zadziałały z powodu błędu konfiguracji), 
	- zabezpiecza przed nieoczekiwanym `permit`,
	- występuje zawsze na końcu typu account.
- `pam_permit.so`:
	- jawne zakończenie sukcesem, nawet jeśli poprzednie moduły nie dały jednoznacznego `OK`,
	- stanowi zagrożenie, bo może przepuścić nieautoryzowany dostęp, 
	- stosowane tylko w wyjątkowych sytuacjach (np. środowiska testowe), 
	- w bezpiecznej konfiguracji `common-account` nie powinien występować.

## Flagi kontrolne w pliku common-account
W pliku common-account niemal zawsze moduły występują z flagą `required`, ponieważ:
- polityka musi być egzekwowana,
- `required` pozwala na przejście przez cały stos `account` i tym samym umożliwia zapis zdarzeń przez wszystkie moduły, co ma znaczenie audytowe.
Wyjątkiem są moduły `pam_nologin.so` i `pam_deny.so`, które powinny występować z flagą `requisite`.

## Zagrożenia związane z plikiem common-account
- `pam_nologin.so` - to istotny moduł, który uniemożliwia dostęp zwykłym użytkownikom dostęp do systemu w momencie, kiedy system może być wrażliwy (np. jest w trybie maintenance). Odmowa dostępu nie dotyczy użytkownika root. Dobrą praktyką jest umieszczanie tego modułu  na początku, żeby zabezpieczyć system gdy jest w trybie maintenance. Brak tego modułu, może stwarzać zagrożenie dla systemu podczas prac konserwacyjnych.
- brak modułu `pam_faillock.so` oznacza, że liczenie nieudanych prób logowania odbywa się bez egzekwowania blokady. To czerwona flaga, na którą należy zawsze reagować.
- `pam_unix.so` - brak tego modułu oznacza brak sprawdzenia:
	- czy konto jest zablokowane w `/etc/shadow`,
	- czy konto jest wygasłe,
	- czy konto istnieje.   
	To poważny błąd, który może dać dostęp nieuprawnionym użytkownikom.
- brak modułu `pam_sss.so` oznacza, że domena nie egzekwuje polityk.
- brak modułu `pam_deny.so` to potencjalny bypass.
- obecność `pam_permit.so` stanowi zagrożenie, ponieważ może przepuścić nieautoryzowany dostęp. W bezpiecznej konfiguracji nie powinien występować,  
- niewłaściwe flagi stanowią inny typ zagrożenia, ponieważ mogą powodować pomijanie pozostałych modułów typu account w całym stosie PAM. W tym typie nie powinny się znajdować flagi `sufficient` i `optional`.

## Wnioski
- Moduł `pam_nologin.so` zgodnie z dobrą praktyką powinien być na początku typu, czyli od razu powinien sprawdzać tryb maintenance i w razie jego wystąpienia powodować natychmiastowe przerwanie typu `account` i zwrócenie niepowodzenia dzięki fladze `requisite`. Ale uwaga, są wyjątki: czasami administratorzy idą na pewne kompromisy, umieszczając ten moduł dopiero za modułami ochronnymi i uwierzytelniającymi. Ma to na celu pozostawienie w logach śladu, jakie konto próbowało się dostać do systemu w trakcie trwania prac np. konserwacyjnych. Więc nie zawsze umiejscowienie tego modułu w innym miejscu niż pierwsze jest błędem. Ważne, żeby nie znajdował się na końcu typu. Więc to co należy sprawdzać to czy i gdzie ten moduł się znajduje oraz czy to jest świadomy kompromis czy błąd. 
- moduł `pam_faillock.so` powinien zawsze być obecny, w przeciwnym razie nie zostanie sprawdzone czy konto zostało zablokowane z powodu zbyt dużej ilości nieudanych prób. Brak tego modułu to zawsze czerwona flaga, wymagająca sprawdzenia i korekty.
- brak modułu `pam_unix.so` jest również bardzo niebezpieczne, gdyż bez niego nie zostanie sprawdzony status konta lokalnego (czy istnieje, nie jest zablokowane, czy nie wygasło). Należy więc zawsze sprawdzać czy moduł występuje w pliku i czy jest we właściwym miejscu. 
- moduł `pam_sss.so` to analogiczna sytuacja do modułu unix, tym wyjątkiem, że sss sprawdza użytkowników domenowych a nie lokalnych.
- brak modułu `pam_deny.so` może stanowić potencjalny bypass, zawsze powinien znajdować się po modułach ochronnych i uwierzytelniających ale przed modułem permit. Moduł ten wymusza niepowodzenie typu `account`, jeśli wcześniejsze moduły nie zakończyły się jednoznacznym sukcesem. 
- brak modułu `pam_permit.so` - w bezpiecznej konfiguracji `common-account` moduł ten nie powinien występować. Jego obecność stanowi ryzyko, ponieważ jawnie przyznaje dostęp, nawet jeśli poprzednie moduły nie dały jednoznacznego sukcesu. Stosowany tylko w środowiskach testowych. 
- należy także sprawdzać czy moduły mają właściwe flagi, wszystkie powinny być ustawione na required poza modułami nologin i deny, które występują zawsze z flagą `requisite`. Należy również zwracać uwagę czy rozszerzone flagi kontrolne (jeśli występują) nie pomijają istotnych modułów. 

## Przykłady analizy konfiguracji pliku common-account (Case Study)

### Case study 1  
```
account requisite pam_nologin.so    
account required pam_unix.so    
account required pam_faillock.so    
account requisite pam_deny.so    
```
Analiza:   
Wszystkie moduły tutaj mają właściwą flagę, jednak kolejność modułów `pam_unix.so` i `pam_faillock.so` nie są poprawne, Dobra praktyka wymaga sprawdzenia czy moduł `pam_faillock.so` nie zablokował konta ze względu na zbyt dużą ilość nieudanych prób, które mogą świadczyć o brute force. W tym przypadku konto może przejść część walidacji a dopiero potem zostać zablokowane ze względu na ilość prób. To nie jest twardy bypass, nie stanowi też jednoznacznego zagrożenia, jednak ponieważ jest to mało czytelne może prowadzić do edge-case-ów.  
  
Wniosek:  
Nieoczekiwane zagrożenie, konfiguracja niezalecana, należy ją poprawić zmieniając kolejność modułów unix i faillock. 

### Case study 2
```
account required pam_faillock.so   
account sufficient pam_unix.so   
account required pam_sss.so    
account requisite pam_deny.so    
```
Analiza:  
Ten przypadek wymaga odrobinę większego zrozumienia, ponieważ na pierwszy rzut oka, konsekwencje takiego zapisu nie są oczywiste. To, co jest tutaj oczywiste to:
- brak modułu `pam_nologin.so`  w systemach z interaktywnymi logowaniami stanowi zagrożenie dla systemu gdy jest w trybie maintenance,
- moduł `pam_faillock.so` jest ustawiony we właściwym miejscu i ma poprawnie ustawioną flagę,
- flaga modułów `pam_sss.so` i `pam_deny.so` są ustawione poprawnie, podobnie jak kolejność tych modułów w zapisie. 
Jedynym złym elementem w tym zapisie jest flaga `sufficient` dla modułu `pam_unix.so`. To oczywiste, że jest to zła flaga, lecz nie tak oczywiste są powody dlaczego ta flaga stwarza zagrożenie. Otóż, plik globalny `common-account` zwykle jest jedynie częścią konfiguracji PAM dołączaną do plików konfiguracji PAM dla aplikacji. `Sufficient` w module `pam_unix.so` powoduje pominięcie wszystkich (nie tylko w ujętych w pliku common) modułów typu account w danym stosie PAM dla tej aplikacji. To oznacza, że jeśli aplikacja poza dołączonym plikiem `common-account` ma jeszcze jakieś inne moduły, to nie zostaną one wykonane, jeśli znajdują się w stosie za modułem `pam_unix.so` z flagą `sufficient`, gdyż w momencie gdy moduł `pam_unix.so` zwróci sukces, typ zostanie zakończony. Konsekwencje takiego zapisu:
- samo pominięcie modułów za unix, gdyby nie było więcej zapisów w pliku PAM aplikacji nie stwarza zagrożenia,
- gdyby jednak poza plikiem common było ich więcej , to konto lokalne nie byłoby już w żaden więcej sposób sprawdzane typem `account`,
- lokalni użytkownicy  nie muszą przechodzić przez całą drogę walidacji jak użytkownicy domenowi,
- dodane w przyszłości moduły nie będą dotyczyć kont lokalnych.  
  
Wniosek:   
Na plik `common-account` należy patrzeć szeroko z perspektywy systemu a nie jednego pliku. `Sufficient` nie powinien się znaleźć przy żadnym module w tym typie. Trzeba przewidywać przyszłe modyfikacje i rozwój polityk PAM. Flagę przy module `pam_unix.so` należy zmienić na `required`.  

### Case study 3
```
account required pam_unix.so  
account required pam_faillock.so  
account requisite pam_nologin.so  
account requisite pam_deny.so  
account required pam_permit.so  
```
Analiza:   
Wszystkie flagi modułów są poprawne lecz kolejność nie jest optymalna, jednakże:
- moduł `pam_faillock.so` za modułem `pam_unix.so` a powinien być przed nim. Konto, które jest zablokowane po przekroczeniu dozwolonych prób najlepiej blokować od razu.
- moduł `pam_nologin.so` w tym przypadku wygląda jak świadomy kompromis, po to by moduły `pam_unix.so` i `pam_faillock.so` zapisały się w logach. To nie zagraża systemowi i choć nie jest zalecane to takie kompromisy są czasami stosowane - więc nie należy panikować.  
- moduł `pam_permit.so` nie jest koniecznością, nie stanowi też bezpośredniego zagrożenia, jednak w bezpiecznej konfiguracji `common-account` nie powinien występować. 
  
Wniosek:   
Dobrze jest w tym przypadku upewnić się czy kompromis związany z `pam_nologin.so` jest świadomy oraz zasugerować zmianę kolejności między modułami `pam_unix.so` i `pam_faillock.so` dla lepszej czytelności. To jest raczej kosmetyka niż błąd lub zagrożenie. 