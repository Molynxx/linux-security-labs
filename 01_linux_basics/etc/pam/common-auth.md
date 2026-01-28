# etc/pam.d/common-auth

## Cel 
Celem tego laboratorium jest poznanie pliku common-auth oraz zrozumienie jak powinna wyglądać bezpieczna konfiguracja tego pliku.

## Czym jest /etc/pam.d/common-auth
Common-auth odpowiada za proces uwierzytelniania i zawiera ustawienia globalne PAM, czyli zestaw typowych operacji, które są wspólne dla wszystkich plików PAM dla aplikacji. By załączyć plik common-auth do pliku PAM aplikacji, należy użyć polecenia @include common-auth.

# Typowe moduły i ich kolejność dla fazy auth
Znajdują się tutaj moduły ochronne, uwierzytelniające oraz decyzyjne:
- Moduły ochronne: 
	- pam_faillock.so / pam_tally2.so - moduły odpowiadające za liczenie prób (blokada brute force),
	- pam_faildelay.so - modów ten wprowadza opóźnienie po nieudanej próbie logowania, nie jest obowiązkowy jeśli występuje faillock/tally2, lecz jest zalecany ponieważ zmniejsz ryzyko brute force.
- Moduły uwierzytelniające:
	- pam_unix.so - moduł lokalnego logowania,
	- pam_ldap.so. i pam_sss.so - moduły sprawdzania użytkownika  na serwerze.
- Moduły decyzyjne:
	- pam_deny.so - moduł kończący stos auth wymuszając końcową decyzję 'deny' (odmów),
	- pam_permit.so - moduł kończący stos auth wymuszając końcową decyzję 'allow' (zezwól).

## Flagi kontrolne dla modułów znajdujących się w common-auth
W fazie auth zalecanym standardem jest używanie flagi 'required' dla modułów ochronnych, uwierzytelniających oraz decyzyjnych, ponieważ pozwala to na wykonanie pełnego stosu PAM bez ujawniania punktu porażki i minimalizuje ryzyko obejścia polityki bezpieczeństwa. POza typowymi flagami (required, requisite, sufficient, optional) mogą tutaj wystąpić rozszerzone flagi kontrolne, które pozwalają na większą elastyczność w regulowaniu działania modułów. Przykład:
auth required [succes=1 default=ignore] pam_unix.so
W tym przypadku succes=1 oznacza, że jeśli moduł zwróci sukces to jedna kolejna linia zostanie pominięta. To może być potrzebne w takim przypadku, gdy najpierw jest moduł logowania lokalnego pam_unix.so a zaraz potem moduł sprawdzający użytkownika na serwerze pam_sss.so. Jeśli logowanie lokalne się powiedzie, to sprawdzanie danych na serwerze nie jest już potrzebne. 

## Na co zwracać uwagę
- czy były jakieś zmiany w pliku common-auth, jeśli tak to jakie, kiedy i przez kogo wprowadzone,
- czy w pliku są wszystkie moduły zapewniające bezpieczeństwo procesu logowania,
- czy jest właściwa kolejność modułów, 
- czy moduły mają właściwą flagę kontrolną,
- czy w rozszerzonych flagach kontrolnych dla odpowiedzi modułu success=n, n nie pomija jakiegoś ważnego modułu. 

## Wnioski 
- Każda zmiana pliku common-auth to potencjalne zagrożenie, ale nie koniecznie oznacza incydent. Dlatego zawsze należy sprawdzać co, kiedy i przez kogo zostało zmienione, 
- Brak istotnego dla bezpieczeństwa modułu, np. brak modułów ochronnych stanowi duże ryzyko. Brak modułów ochronnych zwiększa zagrożenie brute force. 
- Moduły uwierzytelniające powinny występować po modułach ochronnych, kolejność jest tutaj bardzo istotna, ponieważ moduły uwierzytelniania przed modułami ochronnymi nie są w żaden sposób zabezpieczone przed brute force a moduły ochronne występujące po nich są bezużyteczne, 
- Należy zawsze sprawdzać czy moduły zwarte w pliki common-auth mają odpowednie flagi kontrolne. Zalecane flagi dla tego typu to required lub opcjonalnie rozszerzone flagi kontrolne. Jednak gdy występuje rozszerzona flaga kontrolna należy sprawdzić czy nie przeskakuje ona istotnych w procesie uwierzytelniania modułów. Należy pamiętać, że moduły znajdujące się w plikach PAM dla aplikacji wraz z załączonym plikiem common-auth tworzą stos, a rozszerzone flagi kontrolne przeskakujące moduły należy analizować w kontekście całego stosu PAM. 
- Zmiana flagi z required na sufficient może stanowić baypass, a zmiana modułów na uwierzytelniających na optional może powodować nieograniczony dostęp do systemu. 

## Przykłady analizy konfiguracji (case study) 
### Przykład 1
auth required pam_faollock.so preauth
auth [succes=1 default=ignore] pam unix.so
auth sufficient pam_sss.so 
auth requisite pam_deny.so
auth required pam_permit.so
Analiza:
- Moduł ochronny i moduł uwierzytelniania są  ustawione poprawnie, w przypadku udanego logowania lokalnego zostanie pominięty moduł pam_sss.so, co jest logiczne, ponieważ skoro powiodło się logowanie  lokalne nie ma już potrzeby sprawdzać danych logowania na serwerze.
- Zagrożeniem (choć nieoczywistym) jest flaga modułu pam_sss.so. Sufficient w przypadku gdy moduł zwróci sukces to sprawdzanie typu auth się zakończy i pozostałe moduły się nie wykonają, więc mamy do czynienia z baypass modułu pam_deny.so.
Wniosek: Konfiguracja w takim kształcie jest niezalecana i niebezpieczna, moduł pam_sss.so powinien mieć flagę required. 

### Przykład 2
auth sufficient pam_unix.so
auth required pam_faillock.so
auth required pam_deny.so
auth optional pam_permit.so 
Analiza:
- Moduł uwierzytelniający jest przed modułem ochronnym, więc uwierzytelnianie odbywa się bez jakiejkolwiek ochrony przed brute force. 
- Moduł uwierzytelniający ma flagę sufficient, a to oznacza, że w momencie kiedy moduł zwróci sukces wszystkie pozostałe modułu typu auth zostaną pominięte. 
Wniosek: To jest krytycznie zła konfiguracja pliku narażona na brute force i gwarantuje dostęp po złamaniu hasła. Taka konfiguracja pliku nie powinna nigdy występować. 

### Przykład 3
auth required pam_faillock.so preauth
auth [success=2 default=ignore] pam_unix.so
auth required pam_sss.so
auth requisite pam_deny.so
auth required pam_permit.so
Analiza:
- Moduły są w poprawnej kolejności, występują moduły ochronne. 
- Moduł uwierzytelniający ma rozszerzoną flagę kontrolną co powoduje, że gdy moduł zwróci sukces nastąpi pominięcie 2 kolejnych linii. Taki zapis ma logiczny sens, ponieważ w przypadku powodzenia podczas uwierzytelniania lokalnego nie ma potrzeby uruchamiać modułu uwierzytelniającego na serwerze ani modułu pam_deny.so. Naturalnym działaniem jest w tym przypadku przeskok do modułu wymuszającego końcową decyzję 'allow'.
Wniosek: Konfiguracja pliku jest w tym przypadku poprawna i bezpieczna, choć można zwiększyć bezpieczeństwo dodając moduł pam_faildelay.so po pierwszym module ochronnym. 