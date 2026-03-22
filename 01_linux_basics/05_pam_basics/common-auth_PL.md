# etc/pam.d/common-auth

## Cel 
Celem tego laboratorium jest poznanie pliku common-auth oraz zrozumienie jak powinna wyglądać bezpieczna konfiguracja tego pliku.

## Czym jest /etc/pam.d/common-auth
Common-auth odpowiada za proces uwierzytelniania i zawiera ustawienia globalne PAM, czyli zestaw typowych operacji, które są wspólne dla wszystkich plików PAM dla aplikacji. By załączyć plik common-auth do pliku PAM aplikacji, należy użyć polecenia `@include common-auth`.

## Typowe moduły i ich kolejność dla fazy auth
Znajdują się tutaj moduły ochronne, uwierzytelniające oraz decyzyjne:
- Moduły ochronne: 
	- `pam_faillock.so / pam_tally2.so` - liczenie prób (blokada brute force),
	- `pam_faildelay.so` - opóźnienie po nieudanej próbie logowania, 
	- `pam_unix.so` - moduł lokalnego logowania,
	- `pam_ldap.so` i `pam_sss.so` - moduły logowania zdalnego,
- Moduły decyzyjne:
	- `pam_deny.so` - wymusza końcową decyzję `deny`,
	- `pam_permit.so` - jeśli wszystko się powiodło, decyduje o przyznaniu dostępu `allow`.

## Flagi kontrolne dla modułów znajdujących się w common-auth
W tym typie najbezpieczniejszym wyborem jest stosowanie flagi `required`. Wyjątkiem jest moduł `pam_deny.so`, który może mieć flagę `requisite`, aby natychmiast przerwać proces auth w przypadku niepowodzenia. Mogą tutaj występować rozszerzone flagi kontrolne:  
Przykład:  
	`auth required [success=1 default=ignore] pam_unix.so`  
W tym przypadku `success=1` oznacza, że jeśli moduł zwróci sukces to jedna kolejna linia zostanie pomięta. 

## Na co zwracać uwagę
- zmiany w pliku common-auth, jeśli były to jakie, kiedy i przez kogo wprowadzone,
- czy w pliku są wszystkie moduły zapewniające bezpieczeństwo procesu logowania,
- czy jest właściwa kolejność modułów, 
- czy moduły mają właściwą flagę kontrolną,
- czy w rozszerzonych flagach kontrolnych dla odpowiedzi modułu `success=n`, `n` nie pomija jakiegoś ważnego modułu. 

## Wnioski 
- każda zmiana pliku common-auth to potencjalne zagrożenie, jednak nie zawsze oznacza incydent,
- brak modułów ochronnych zwiększa zagrożenie brute force, 
- Moduły uwierzytelniające powinny występować po modułach ochronnych, aby moduły ochronne mogły działać poprawnie,  
- należy sprawdzać czy moduły mają odpowiednie flagi kontrolne. Niewłaściwe flagi mogą pomijać istotne moduły w stosie typu. 

## Przykłady analizy konfiguracji (case study) 

### Przykład 1  

auth sufficient pam_unix.so    
auth required pam_faillock.so    
auth required pam_deny.so    
auth optional pam_permit.so    

Analiza:    
- Moduł uwierzytelniający jest przed modułem ochronnym, więc uwierzytelnianie odbywa się bez jakiejkolwiek ochrony przed brute force. 
- Moduł uwierzytelniający ma flagę `sufficient`, a to oznacza, że w momencie kiedy moduł zwróci sukces wszystkie pozostałe modułu typu auth zostaną pominięte.        
To jest krytycznie zła konfiguracja pliku narażona na brute force i gwarantuje dostęp po złamaniu hasła.  

Sugerowane działania:  
- zmienić flagę modułu `pam_unix.so` na `required`,
- umieścić moduł `pam_unix.so` za modułem ochronnym. 

### Przykład 2

auth required pam_faillock.so preauth    
auth [success=2 default=ignore] pam_unix.so    
auth required pam_sss.so    
auth requisite pam_deny.so    
auth required pam_permit.so    

Analiza:    
- moduły są w poprawnej kolejności, występują moduły ochronne. 
- moduł uwierzytelniający ma rozszerzoną flagę kontrolną co powoduje, że gdy moduł zwróci sukces nastąpi pominięcie 2 kolejnych linii. Ustawienie jest poprawne.

Sugerowane działania:   
- konfiguracja jest poprawna i nie wymaga działań. 