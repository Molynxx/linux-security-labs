# /etc/pam.d/login - PAM dla usługi login

## Cel laboratorium
Celem tego laboratorium jest zrozumienie działania ustawień PAM dla aplikacji login.

## Czym jest /etc/pam.d/login
To plik konfigurujący PAM dla aplikacji login, znajdującej się w /bin/login. Tu należy podkreślić czym jest aplikacja login. Login jest mechanizmem lokalnego logowania do systemu Linux, używanym przy dostępie do konsoli (TTY). Odpowiada za pełny proces logowania:
- uwierzytelnienie użytkownika,
- sprawdzenie systemu i konta, 
- utworzenie pełnej sesji użytkownika.  
Ta aplikacja odpowiedzialna jest za podstawowe, niskopoziomowe zabezpieczenie sytemu, które zabezpiecza lokalny dostęp do systemu i działa niezależnie od SSH i GUI. Podczas gdy SSH umożliwia jedynie dostęp zdalny do systemu, login robi o wiele więcej:
- pyta o użytkownika, 
- sprawdza hasło (przez PAM -> /etc/shadow),
- ustawia pełną sesję,
- ustawia UID, GID, grupy,
- ustawia środowisko,
- rejestruje sesję,
- obsługuje SELinux - czyli kontekst bezpieczeństwa logowania i sesji,
- reaguje na tryb maintenance (konserwacji).

## Jakie zastosowanie ma plik /etc/pam.d/login
Plik ten jest zbiorem instrukcji jak aplikacja powinna działać w zgodzie z PAM, są w nim zdefiniowane sposoby obsługiwania modułów standardowych. W pliku tym mogą być załączone pliki globalne (common), które stanowią zbiór wspólnych zasad dla aplikacji oraz indywidualne modyfikacje działania modułów dla aplikacji login. 

## Kolejność wywoływania typów przez login
Login wywołuje typy w następującej kolejności:
- auth,
- account,
- session,
- password - tylko przy zmianie hasła. 

## Jak wyglądają zapisy w /etc/pam.d/login
Zapisy we wszystkich plikach PAM wyglądają podobnie, choć ich zawartość może się różnić. Ale zawsze mają ten sam układ:

[typ] [flaga kontrolna] [nazwa modułu] [opcje]

Przykładowy zapis w pliku PAM login:  
session [success=ok ignore=ignore module_unknown=ignore default=bad] pam_selinux.so. 

- typ session - dotyczy otwierania sesji,
- flaga - tu rozszerzona flaga kontrolna, czyli zamiast required, requisite, sufficient, optional występuje:  
	- success=ok:  
		- SELinux istnieje,  
		- jest włączony,  
		- odpowiada,   
		to oznacza, że wszystko jest w porządku, sesja nie jest blokowana,  
	- ignore=ignore:  
		- SELinux jest wyłączony,  
		- sesja go nie dotyczy,  
		to oznacza, ze wszystko jest w porządki, sesja może być kontynuowana,  
	- module_unknown=ignore:   
		- SELinux nie istnieje w systemie (nie jest zainstalowany),
		- dystrybucja nie używa SELinyx,    
		ten zapis nie oznacza błędu, mówi, że jeśli SELinux nie istnieje w systemie to należy go pominąć, żeby nie nastąpiła blokada sesji,  
	- default=bad:  
		- wszystko inne niż powyższe oznacza błąd, wskazuje, że moduł nie działa poprawnie, że istnieją błędy np.:  
			- błąd inicjalizacji,    
			- uszkodzona konfiguracja SELinux,  
			- wewnętrzny błąd modułu,  
			- brak dostępu do polityk.  
		W takim przypadki uznawane jest to za niebezpieczne i sesja może zostać przerwana. Rozszerzone flagi są stosowane w module pam_selinux.so wyłącznie w typie session. Tego rodzaju zapisy z rozszerzonymi flagami kontroli pomagają na większą elastyczność, ponieważ required nie rozróżnia 'nie ma' od 'nie działa'.  

## Kolejność modułów w obrębie jednego typu
Kolejności modułów w obrębie jednego typie ma ogromne znaczenie.
- moduły typu auth - tu znajdują się moduły wprowadzające opóźnienia po nieudanych próbach, moduły pobierające dane uwierzytelniające, moduły zliczające ilość nieudanych prób.   
- moduły typu account - tym odpowiada za autoryzacje, więc znajdują się tutaj modułu sprawdzające czy użytkownik może się zalogować, jako pierwszy z flagą requisite jest zalecany moduł pam_nologin.so - który sprawdza czy system nie jest w trakcie trwania prac serwisowych. Następnie sprawdzane są dane z modułów, które brały udział w uwierzytelnieniu oraz moduły sprawdzające czy konto nie jest zablokowane w systemie,
- moduły typu session odpowiadające za:
	- ustalenie kontekstu bezpieczeństwa pam_SELinux.so,
	- ustawienie środowiska pam_env.so,
	- ustawienie limitóe pam_limits.so,
	- ustawienie UID sesji pam_loginuid.so,
	- ustawienia grup, mail, motd, itd,
- załączone pliki common - na końcu.

## pam_nologin.so
To istotny moduł, który warto zrozumieć lepiej. Moduł ten może być stosowany w plikach konfiguracji PAM dla każdej aplikacji. To jest moduł sprawdzający czy system nie jest w trybie maitenance. Administrator systemu, prowadząc prace serwisowe lub naprawcze może potrzebować spokoju w systemie. Nologin pozwala mu ograniczyć dostęp do systemu dla wszystkich użytkowników poza root, żeby jego działania mogły zakończyć się bezpiecznie. Jeśli plik PAM dla aplikacji ma zapis: account requisite pam_nologin.so to w pierwszej kolejności zostanie sprawdzone czy system nie jest w trybie maintenance. Jeśli tak, żaden użytkownik się nie zaloguje. Sprawdzenie czy system jest w stanie maintenance polega na sprawdzeniu czy istnieje plik /etc/nologin. Jak to działa: Administrator tworzy plik tekstowy, który wyświetli na ekranie użytkownika komunikat, na przykład 'system w trakcie prac serwisowych'. By stworzyć taki plik wystarczy, że użytkownik posiadający uprawnienia sudo posłuży się takim poleceniem:  
	sudo sh -c "echo 'System w trakcie prac serwisowych' > /etc/nologin"  
Po zakończeniu prac serwisowych plik /etc/nologin jest usuwany i system wraca do normalnej pracy. Uwaga: nologin uniemożliwia zalogowanie się zwykłym użytkownikom, ale nie przerywa sesji trwających w czasie powstania pliku nologin. Plik może zostać usunięty tylko przez zalogowanego użytkownika z uprawnieniami sudo lub root. 

## Red flags SOC
- usunięcie modułu pam_nologin.so,
- zmiany w nietypowych/niestandardowych modułach PAM (np.: pam_security.so),
- manipulacje w typie session (zmiana limitów, grup, SELinux, itd),
- zmiany w plikach common,
- login działający gdy system powinien być zamknięty (backdoor).

## Wnioski
- W ustawieniach kolejności modułów, dobra praktyka jest wpisywanie w plikach konfiguracyjnych PAM dla aplikacji w pierwszej kolejności specyficznych dla danej aplikacji a dopiero później globalnych (common).
- Bardzo ważne jest sprawdzanie kto i kiedy zmienił plik nologin w katalogu /etc/ za pomocą polecenia stat (mtime, ctime, acces).
- Należy sprawdzać czy moduły nologin są poprawnie ustawione w plikach PAM, czy nie zostały usunięte oraz czy moduł pam_nologin.so nie został usunięty.
- Należy sprawdzać metadane plików common oraz PAM dla aplikacji, a także treść tych plików. 





