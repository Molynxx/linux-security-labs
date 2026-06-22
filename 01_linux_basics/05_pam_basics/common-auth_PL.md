# etc/pam.d/common-auth

## Cel 
Celem tego laboratorium jest poznanie pliku common-auth oraz zrozumienie jak powinna wyglądać bezpieczna konfiguracja tego pliku.

## Czym jest /etc/pam.d/common-auth
Common-auth odpowiada za proces uwierzytelniania i zawiera ustawienia globalne PAM, czyli zestaw typowych operacji, które są wspólne dla wszystkich plików PAM dla aplikacji. By załączyć plik common-auth do pliku PAM aplikacji, należy użyć polecenia @include common-auth.

## Typowe moduły i ich kolejność dla fazy auth
Znajdują się tutaj moduły ochronne, uwierzytelniające oraz decyzyjne:
 - Moduły uwierzytelniające:
	- `pam_unix.so` - moduł lokalnego logowania,
	- `pam_ldap.so` i `pam_sss.so` - moduły sprawdzania użytkownika  na serwerze.
- Moduły ochronne: 
	- `pam_faillock.so` / `pam_tally2.so` - moduły odpowiadające za liczenie prób (blokada brute force),
	- `pam_faildelay.so` - moduł ten wprowadza opóźnienie po nieudanej próbie logowania, nie jest obowiązkowy jeśli występuje faillock/tally2, lecz jest zalecany ponieważ zmniejsza ryzyko brute force.
- Moduły decyzyjne:
	- `pam_deny.so` - moduł kończący stos auth wymuszając końcową decyzję 'deny' (odmów), jeśli wcześniejsze moduły nie zwróciły jednoznacznej decyzji,
	- `pam_permit.so` - moduł zwracający sukcess nawet jeśli poprzednie moduły nie dały jednoznacznego sukcesu. Moduł ten nie powinien być stosowany w systemach produkcyjnych, ponieważ może przepuścić użytkownika bez uwierzytelnienia.

## Flagi kontrolne dla modułów znajdujących się w common-auth
W fazie auth zalecanym standardem jest używanie flagi `required` dla modułów ochronnych, uwierzytelniających oraz decyzyjnych, ponieważ pozwala to na wykonanie pełnego stosu PAM bez ujawniania punktu porażki i minimalizuje ryzyko obejścia polityki bezpieczeństwa. Poza typowymi flagami (`required`, `requisite`, `sufficient`, `optional`) mogą tutaj wystąpić rozszerzone flagi kontrolne, które pozwalają na większą elastyczność w regulowaniu działania modułów.   
Przykład:   
	`auth required [success=1 default=ignore] pam_unix.so` 
W tym przypadku `success=1` oznacza, że jeśli moduł zwróci sukces to jedna kolejna linia zostanie pominięta. To może być potrzebne w takim przypadku, gdy najpierw jest moduł logowania lokalnego `pam_unix.so` a zaraz potem moduł sprawdzający użytkownika na serwerze `pam_sss.so`. Jeśli logowanie lokalne się powiedzie, to sprawdzanie danych na serwerze nie jest już potrzebne. 

## Na co zwracać uwagę
- czy były jakieś zmiany w pliku `common-auth`, jeśli tak to jakie, kiedy i przez kogo wprowadzone,
- czy w pliku są wszystkie moduły zapewniające bezpieczeństwo procesu logowania,
- czy jest właściwa kolejność modułów, 
- czy moduły mają właściwą flagę kontrolną,
- czy w rozszerzonych flagach kontrolnych dla odpowiedzi modułu `success=n`, n nie pomija jakiegoś ważnego modułu. 

## Wnioski 
- Każda zmiana pliku common-auth to potencjalne zagrożenie, ale nie koniecznie oznacza incydent. Dlatego zawsze należy sprawdzać co, kiedy i przez kogo zostało zmienione, 
- Brak istotnego dla bezpieczeństwa modułu, np. brak modułów ochronnych stanowi duże ryzyko. Brak modułów ochronnych zwiększa zagrożenie brute force. 
- Moduły uwierzytelniające powinny występować po modułach ochronnych, kolejność jest tutaj bardzo istotna, ponieważ moduły uwierzytelniania przed modułami ochronnymi nie są w żaden sposób zabezpieczone przed brute force a moduły ochronne występujące po nich są bezużyteczne, 
- Należy zawsze sprawdzać czy moduły zwarte w pliku `common-auth` mają odpowiednie flagi kontrolne. Zalecane flagi dla tego typu to required lub opcjonalnie rozszerzone flagi kontrolne. Jednak gdy występuje rozszerzona flaga kontrolna należy sprawdzić czy nie przeskakuje ona istotnych w procesie uwierzytelniania modułów. Należy pamiętać, że moduły znajdujące się w plikach PAM dla aplikacji wraz z załączonym plikiem `common-auth` tworzą stos, a rozszerzone flagi kontrolne przeskakujące moduły należy analizować w kontekście całego stosu PAM. 
- Zmiana flagi z `required` na `sufficient` może stanowić bypass, a zmiana modułów  uwierzytelniających na optional może powodować nieograniczony dostęp do systemu. 

## Przykłady analizy konfiguracji (case study) 
Poniższe przykłady zawierają użycie opcji dla modułu `pam_faillock.so`, warto tutaj wspomnieć czym te opcje są.   
	Moduł `pam_faillock.so` występuje zwykle trzykrotnie: z opcją `preauth`, która sprawdza czy konto nie jest już zablokowane przed uwierzytelnieniem, opcją `authfail`, która zlicza nieudane próby po błędnym haśle oraz z opcją `authsucc`, która zeruje licznik nieudanych prób po udanym zalogowaniu. Wszystkie opcje pełnią różne role i muszą być użyte w osobnych liniach PAM.  
	W przypadku gdy polityka modułu jest określona w pliku konfiguracyjnym modułu `/etc/security/faillock.conf` nie ma konieczności używania opcji, ale trzeba zawsze najpierw zweryfikować ustawienia pliku `.conf`.

### Case study 1
```
auth required pam_faillock.so preauth  
auth [success=1 default=ignore] pam_unix.so  
auth sufficient pam_sss.so   
auth requisite pam_deny.so  
auth required pam_permit.so  
```
Analiza:  
- Moduł ochronny i moduł uwierzytelniania są ustawione poprawnie, w przypadku udanego logowania lokalnego zostanie pominięty moduł pam_sss.so (`success=1` - pomija 1 kolejną linię), co jest logiczne, ponieważ skoro powiodło się logowanie lokalne nie ma już potrzeby sprawdzać danych logowania na serwerze. 
- brak modułu `pam_faillock.so` z opcją `authfail` za modułem uwierzytelniającym, która będzie zliczać próby. Bez niej moduł `pam_faillock.so` z opcją `preauth` nie będzie miał danych, by sprawdzić czy nie przekroczono nieudanych prób. Ponadto, po module z opcją `authfail`, należy jeszcze raz umieścić moduł, tym razem z opcją `authsucc`, aby wyzerować licznik nieudanych prób po udanym zalogowaniu, 
- moduł `pam_sss.so` - zakończy realizację procesu PAM, co jest zasadne ponieważ skoro powiodło się zdalne logowanie to za cały dalszy proces odpowiada domena nie system lokalny, 
- moduł `pam_permit.so` nie powinien być używany w systemach produkcyjnych.

Sugerowane działania:  
- dodać moduł `pam_faillock.so` z opcją `authfail` po module `pam_unix.so`,
- dodać moduł `pam_faillock.so` z opcją `authsucc` po module z opcją `authfail`, aby zerować licznik po udanym logowaniu, 
- usunąć moduł `pam_permit.so`.

### Case study 2
```
auth sufficient pam_unix.so  
auth required pam_faillock.so  
auth required pam_deny.so  
auth optional pam_permit.so  
```
Analiza:  
- moduł uwierzytelniający ma flagę sufficient, a to oznacza, że w momencie kiedy moduł zwróci sukces wszystkie pozostałe modułu typu auth zostaną pominięte, 
- moduł `pam_faillock.so` nie ma ustawionej opcji, występuje tylko raz, więc w tym przypadku nie robi nic konkretnego,
- moduł `pam_permit.so` nie powinien występować w systemach produkcyjnych.  

Sugerowane działania:  
- zmienić flagę modułu `pam_unix.so` na `required`,
- dodać opcję `authfail` dla modułu `pam_faillock.so`, oraz dodać ten moduł jeszcze dwukrotnie po module `pam_unix.so` z flagami: `authfail` oraz `authsucc`,
- usunąć moduł `pam_permit.so`.

### Case study 3
```
auth required pam_faillock.so preauth  
auth [success=2 default=ignore] pam_unix.so  
auth required pam_sss.so  
auth requisite pam_deny.so  
auth required pam_permit.so  
```
Analiza:  
- moduły są w poprawnej kolejności, 
- żeby mógł się wykonać moduł `pam_faillock.so` z opcją `preauth`, należy dodać ten moduł za `pam_unix.so` z opcją `authfail`, która będzie zliczać próby,
- moduł uwierzytelniający ma rozszerzoną flagę kontrolną co powoduje, że gdy moduł zwróci sukces nastąpi pominięcie 2 kolejnych linii. Ustawienie jest poprawne.
- moduł `pam_permit.so` nie powinien występować w systemach produkcyjnych.

Sugerowane działania:  
- dodać moduł `pam_faillock.so` z opcją `authfail` za modułem `pam_unix.so`, dla poprawnego działania opcji `preauth`,
- usunąć moduł `pam_permit.so`.