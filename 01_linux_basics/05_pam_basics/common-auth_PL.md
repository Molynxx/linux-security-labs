# etc/pam.d/common-auth

## Cel 
Celem tego laboratorium jest poznanie pliku common-auth oraz zrozumienie jak powinna wyglądać bezpieczna konfiguracja tego pliku.

## Czym jest /etc/pam.d/common-auth
Common-auth odpowiada za proces uwierzytelniania i zawiera ustawienia globalne PAM, czyli zestaw typowych operacji, które są wspólne dla wszystkich plików PAM dla aplikacji. By załączyć plik common-auth do pliku PAM aplikacji, należy użyć polecenia `@include common-auth`.

## Typowe moduły i ich kolejność dla fazy auth
Znajdują się tutaj moduły ochronne, uwierzytelniające oraz decyzyjne:
- Moduły uwierzytelniające:   
	- `pam_unix.so` - moduł lokalnego logowania,
	- `pam_ldap.so` i `pam_sss.so` - moduły logowania zdalnego,
- Moduły ochronne: 
	- `pam_faillock.so / pam_tally2.so` - liczenie prób (blokada brute force),
	- `pam_faildelay.so` - opóźnienie po nieudanej próbie logowania, 
- Moduły decyzyjne:
	- `pam_deny.so` - wymusza końcową decyzję `deny`,
	- `pam_permit.so` - jeśli wszystko się powiodło, decyduje o przyznaniu dostępu `allow`.

## Flagi kontrolne dla modułów znajdujących się w common-auth
W tym typie najbezpieczniejszym wyborem jest stosowanie flagi `required`. Wyjątkiem jest moduł `pam_deny.so`, który może mieć flagę `requisite`, aby natychmiast przerwać proces auth w przypadku niepowodzenia. Mogą tutaj występować rozszerzone flagi kontrolne:  
Przykład:  
	`auth required [success=1 default=ignore] pam_unix.so`  
W tym przypadku `success=1` oznacza, że jeśli moduł zwróci sukces to jedna kolejna linia zostanie pominięta. 

## Na co zwracać uwagę
- zmiany w pliku common-auth, jeśli były, to jakie, kiedy i przez kogo wprowadzone,
- czy w pliku są wszystkie moduły zapewniające bezpieczeństwo procesu logowania,
- czy jest właściwa kolejność modułów, 
- czy moduły mają właściwą flagę kontrolną,
- czy w rozszerzonych flagach kontrolnych dla odpowiedzi modułu `success=n`, `n` nie pomija jakiegoś ważnego modułu. 

## Wnioski 
- każda zmiana pliku common-auth to potencjalne zagrożenie, jednak nie zawsze oznacza incydent,
- brak modułów ochronnych zwiększa zagrożenie brute force, 
- Moduły uwierzytelniające powinny występować po modułach ochronnych, aby moduły ochronne mogły działać poprawnie,  
- należy sprawdzać czy moduły mają odpowiednie flagi kontrolne. Niewłaściwe flagi mogą powodować pomijanie istotnych modułów w stosie typu.   

## Przykłady analizy konfiguracji (case study) 

### Przykład 1  

auth sufficient pam_unix.so    
auth required pam_faillock.so    
auth required pam_deny.so    
auth optional pam_permit.so    

Analiza:    
- moduł uwierzytelniający ma flagę `sufficient`, a to oznacza, że w momencie kiedy moduł zwróci sukces wszystkie pozostałe moduły typu auth zostaną pominięte,  
- moduł `pam_faillock.so` nie ma ustawionej opcji,
- moduł `pam_permit.so` - nie powinien występować w systemach produkcyjnych.       
To jest krytycznie zła konfiguracja pliku narażona na brute force i gwarantuje dostęp po złamaniu hasła.  

Sugerowane działania:  
- zmienić flagę modułu `pam_unix.so` na `required`,
- dodać opcję `authfail` dla modułu `pam_faillock.so`,
- usunąć moduł `pam_permit.so`. 

### Przykład 2

auth required pam_faillock.so preauth    
auth [success=2 default=ignore] pam_unix.so    
auth required pam_sss.so    
auth requisite pam_deny.so    
auth required pam_permit.so    

Analiza:    
- żeby mógł zadziałać moduł `pam_faillock.so` z opcją `preauth` należy dodać ten moduł również za `pam_unix.so` z opcją `authfail`, która będzie zliczać próby,
- moduł uwierzytelniający ma rozszerzoną flagę kontrolną co powoduje, że gdy moduł zwróci sukces nastąpi pominięcie 2 kolejnych linii. Ustawienie jest poprawne.
- moduł `pam_permit.so` - nie powinien występować w systemach produkcyjnych. 

Sugerowane działania:   
- dodać moduł `pam_faillock.so` z opcją `authfail`, dla poprawnego działania opcji `preauth`,
- usunąć moduł `pam_permit.so`. 