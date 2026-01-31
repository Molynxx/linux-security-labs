# /etc/pam.d/common-account

## Cel
Celem laboratorium jest zrozumienie czym jest plik /etc/pam.d/common-account oraz zapisów znajdujących się w nim.

## Czym jest /etc/pam.d/common-account
To nie jest plik sprawdzający hasła, to jest plik, który zaczyna egzekwować politykę PAM. Common-account decyduje czy konto może zostać użyte, nawet jeśli uwierzytelnianie przebiegło pomyślnie. Więc nawet gdy hasło jest poprawne lub klucz SSH jest prawidłowy to mimo tego dostęp może być odmówiony. Common-account to warstwa polityk i kontroli, to ustawienia globalne mogące być załączone do plików konfiguracji PAM dla aplikacji. Występuje on zawsze po fazie uwierzytelniania 'auth'.

## Moduły i ich kolejność w pliki common-account 
Wśród modułów występujących w tym pliku wyróżnić można:
- pam_nologin.so:
	- sprawdza czy istnieje plik /etc/nologin (system w trybie maintenance),
	- jeśli istnieje plik nologin, odrzucony zostaje dostęp zwykłych użytkowników do systemu, niezależnie od poprawności uwierzytelnienia. 
pam_faillock.so - to najważniejszy moduł obronny:
	- sprawdza czy konto jest zablokowane,
	- korzysta z danych zebranych w auth.
pam_unix.so - sprawdza:
	- czy konto istnieje, 
	- czy nie jest zablokowane ('!', '*' w pliku /etc/shadow),
	- czy nie wygasło.
	To nie jest uwierzytelnianie, to jest walidacja stanu konta. 
- pam_sss.so - dla kont domenowych:
	- czy konto jest aktywne w AD/LDAP,
	- czy nie zostało wyłączone, 
	- czy spełnia polityki domeny.
-pam_deny.so:
	- wymusza końcową decyzję 'deny',
	- zabezpiecza przed nieoczekiwanym 'permit'.
	Występuje zawsze na końcu, ale przed 'permit'.
-pam_permit.so:
	- jawne zakończenie sukcesem,
	- stosowane po spełnieniu wszystkich warunków.

## Flagi kontrolne w pliku common-account
W pliku common-account niemal zawsze moduły występują z flagą 'required', ponieważ:
- polityka musi być egzekwowana,
- 'required' pozwala na przejście przez cały stos account i tym samym umożliwia zapis zdarzeń przez wszystkie moduły, co ma znaczenie audytowe.
Wyjątkiem są moduły pam_nologin.so i pam_deny.so, które powinny występować z flagą 'requisite'.

## Zagrożenia związane z plikiem common-account
- pam_nologin.so - to istotny moduł, który uniemożliwia dostęp zwykłym użytkownikom dostęp do systemu w momencie, kiedy system może być wrażliwy (np. jest w trybie maintenance). Odmowa dostępu nie dotyczy użytkownika root. Dobrą praktyką jest umieszczanie tego modułu  na początku, żeby zabezpieczyć system gdy jest w trybie maitenance. Brak tego modułu, może stwarzać zagrożenie dla systemu podczas prac konserwacyjnych.
- brak modułu pam_faillock.so oznacza, że liczenie nieudanych prób logowania odbywa się bez egzekwowania blokady. To czerwona flaga, na którą należy zawsze reagować.
- pam_unix.so - brak tego modułu oznacza brak sprawdzenia:
	- czy konto jest zablokowane w /etc/shadow,
	- czy konto jest wygasłe,
	- czy konto istnieje. 
	To poważny błąd, który może dać dostęp nieuprawnionym użytkownikom.
- brak modułu pam_sss.so oznacza, że domena nie egzekwuje polityk.
- brak modułu pam_deny.so to potencjalny bypass.
- brak modułu pam_permit.so to brak domyślnego sukcesu, więc wynik zależy od innych modułów. 
- niewłaściwe flagi stanowią inny typ zagrożenia, ponieważ mogą powodować pomijanie pozostałych modułów typu account w całym stosie PAM. W tym typie nie powinny się znajdować flagi sufficient i optional.

## Wnioski
- Moduł pam_nologin.so zgodnie z dobra praktyką powinien być na początku typu, czyli od razu powinien sprawdzać tryb maintenance i w razie jego wystąpienia powodować natychmiastowe przerwanie typu account i zwrócenie niepowodzenia dzięki fladze requisite. Ale uwaga, są wyjątki: czasami administratorzy idą na pewne kompromisy, umieszczając ten moduł dopiero za modułami ochronnymi i uwierzytelniającymi. Ma to na celu pozostawienie w logach śladu, jakie konto próbowało się dostać do systemu w trakcie trwania prac np. konserwacyjnych. Więc nie zawsze umiejscowienie tego modułu w inny miejscu niż pierwsze jest błędem. Ważne, żeby nie znajdował się na końcu typu. Więc to co należy sprawdzać to czy i gdzie ten moduł się znajduje oraz czy to jest świadomy kompromis czy błąd. 
- moduł pam_faillock.so powinien zawsze być obecny, w przeciwnym razie nie zostanie sprawdzone czy konto zostało zablokowane z powodu zbyt dużej ilości nieudanych prób. Brak tego modułu to zawsze czerwona flaga, wymagająca sprawdzenia i korekty.
- brak modułu pam_unix.so jest również bardzo niebezpieczne, gdyż bez niego nie zostanie sprawdzony status konta lokalnego (czy istnieje, nie jest zablokowane, czy nie wygasło). Należy więc zawsze sprawdzać czy moduł występuje w pliku i czy jest we właściwym miejscu. 
- moduł pam_sss.so to analogiczna sytuacja do moduły unix, tym wyjątkiem ze sss sprawdza użytkowników domenowych a nie lokalnych.
- brak modułu pam_deny.so może stanowić potencjalny bypass, zawsze powinien znajdować się po modułach ochronnych i uwierzytelniających ale przed modułem permit. Moduł ten wymusza niepowodzenie typu account, jeśli wcześniejsze moduły nie zakończyły się jednoznacznym sukcesem. 
- brak modułu pam_permit.so to raczej ryzyko niewłaściwego działania systemu niż ryzyko bezpieczeństwa systemu. Ten moduł powinien być i jawnie udzielać dostępu, pod warunkiem ze wszystkie poprzednie moduły zwróciły sukces. Należy zawsze sprawdzać czy przed modułem permit znajdują się wszystkie konieczne moduły. 
- należy także sprawdzać czy moduły mają właściwe flagi, wszystkie powinny być ustawione na required poza modułami nologin i deny, które występują zawsze z flagą requisite. Należy również zwracać uwagę czy rozszerzone flagi kontrolne (jeśli występują) nie pomijają istotnych modułów. 

## Przykłady analizy konfiguracji pliku common-account (case study)

### Przykład 1
account requisite pam_nologin.so  
account required pam_unix.so  
account required pam_faillock.so  
account requisite pam_deny.so  
Analiza: Wszystkie moduły tutaj mają właściwą flagę, jednak kolejność modułów unix i faillock nie są poprawne, Dobra praktyka wymaga sprawdzenia czy moduł faillock nie zablokował konta ze względu na zbyt dużą ilość nieudanych prób, które mogą świadczyć o brute force. W tym przypadku konto może przejść część walidacji a dopiero potem zostać zablokowane ze względu na ilość prób. To nie jest twardy bypass, nie stanowi też jednoznacznego zagrożenia, jednak ponieważ jest to mało czytelne może prowadzić do edge-case-ów. Ponadto w tym przypadku brak jawnego 'allow' (moduł permit) oznacza, że decyzja opiera się tylko na tym czy któryś moduł się nie powiódł. Może to prowadzić do nieprawnego działania systemu i utrudniać audyt i reasoning.
Wniosek: Nieoczekiwane zagrożenie, konfiguracja niezalecana, należy ją poprawić zmieniając kolejność modułów unix i faillock oraz dodając na końcu moduł permit. 

### Przykład 2
account required pam_faillock.so  
account sufficient pam_unix.so  
account required pam_sss.so  
account requisite pam_deny.so  
Analiza: Ten przypadek wymaga odrobinę większego zrozumienia, ponieważ na pierwszy rzut oka, konsekwencje takiego zapisu nie są oczywiste. To, co jest tutaj oczywiste to:
- brak modułu nologin stanowi zagrożenie dla systemu gdy jest w trybie maitenance,
- moduł faillock jest ustawiony we właściwym miejscu i mam poprawnie ustawioną flagę,
- flaga modułów sss i deny są ustawione poprawnie, podobnie jak kolejność tych modułów w zapisie. 
Jedynym złym elementem w tym zapisie jest flaga sufficient dla modułu unix. To oczywiste, że jest to zła flaga, lecz nie tak oczywiste są powody dlaczego ta flaga stwarza zagrożenie. Otóż, plik globalny common-account zwykle jest jedynie częścią konfiguracji PAM dołączaną do plików konfiguracji PAM dla aplikacji. Sufficient w module unix powoduje pominięcie wszystkich (nie tylko w ujętych w pliku common) modułów typu account w danym stosie PAM dla tej aplikacji. To oznacza, że jeśli aplikacja poza dołączonym plikiem common-account ma jeszcze jakieś inne moduły, to nie zostaną one wykonane, jeśli znajdują się w stosie za modułem unix z flagą sufficient, gdyż w momencie gdy moduł unix zwróci sukces, typ zostanie zakończony. Konsekwencje takiego zapisu:
- samo pominięcie modułów za unix, gdyby nie było więcej zapisów w pliku PAM aplikacji nie stwarza zagrożenia,
- gdyby jednak poza plikiem common było ich więcej , to konto lokalne nie było by już w żaden więcej sposób sprawdzane typem account,
- lokalni użytkownicy  nie muszą przechodzić przez całą drogę walidacji jak użytkownicy domenowi,
- dodane w przyszłości moduły nie będą dotyczyć kont lokalnych.
Wniosek: Na plik common-account należy patrzeć szeroko z perspektywy systemu a nie jednego pliku. Sufficient nie powinien się znaleźć przy żadnym module w tym typie. Trzeba przewidywać przyszło modyfikacje i rozwój polityk PAM. Flagę przy module unix należy zmienić na required. 

### Przykład 3
account required pam_unix.so  
account required pam_faillock.so  
account requisite pam_nologin.so  
account requisite pam_deny.so
account required pam_permit.so  
Analiza: Wszystkie flagi modułów są poprawne lecz kolejność nie jest optymalna, jednakże:
- moduł faillock za modułem unix a powinien być przed nim. Konto, które jest zablokowane po przekroczeniu dozwolonych prób najlepiej blokować od razu.
- moduł nologin w tym przypadku wygląda jak świadomy kompromis, po to by moduły unix i faillock zapisały się w logach. To nie zagraża systemowi i choć nie jest zalecane to takie kompromisy są czasami stosowane - więc nie należy panikować. 
Wniosek: Dobrze jest w tym przypadku upewnić się czy kompromis związany z nologin jest świadomy oraz zasugerować zmianę kolejności między modułami unix i faillock dla lepszej czytelności. To jest raczej kosmetyka niż błąd lub zagrożenie. 