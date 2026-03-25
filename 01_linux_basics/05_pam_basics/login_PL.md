# /etc/pam.d/login - PAM dla usługi login

## Cel laboratorium
Celem tego laboratorium jest zrozumienie działania ustawień PAM dla aplikacji login.

## Czym jest /etc/pam.d/login
To plik konfigurujący PAM dla aplikacji login, znajdującej się w `/bin/login`. Login jest mechanizmem lokalnego logowania do systemu Linux, używanym przy dostępie do konsoli (TTY). Odpowiada za pełny proces logowania:
- uwierzytelnienie użytkownika,
- sprawdzenie systemu i konta, 
- utworzenie pełnej sesji użytkownika.  
Ta aplikacja zapewnia podstawowe, niskopoziomowe zabezpieczenie systemu, które zabezpiecza lokalny dostęp do systemu i działa niezależnie od SSH i GUI. 
Plik ten jest zbiorem instrukcji jak aplikacja powinna działać w zgodzie z PAM, są w nim zdefiniowane sposoby obsługiwania modułów standardowych. W plik ten mogą być załączone pliki globalne (common), które stanowią zbiór wspólnych zasad dla aplikacji oraz indywidualne modyfikacje działania modułów dla aplikacji login. 

## Kolejność wywoływania typów przez login
Login wywołuje typy w następującej kolejności:
- `auth`,
- `account`,
- `session`,
- `password` - tylko przy zmianie hasła. 

## Jak wyglądają zapisy w /etc/pam.d/login
Zapisy we wszystkich plikach PAM wyglądają podobnie, choć ich zawartość może się różnić. Ale zawsze mają ten sam układ:

	[typ] [flaga kontrolna] [nazwa modułu] [opcje]

Przykładowy zapis w pliku PAM login:  
	session [success=ok ignore=ignore module_unknown=ignore default=bad] pam_selinux.so. 

## Kolejność modułów w obrębie jednego typu
Kolejność modułów jest krytyczna dla bezpieczeństwa.
- moduły typu auth - tu znajdują się moduły wprowadzające opóźnienia po nieudanych próbach, moduły pobierające dane uwierzytelniające, moduły zliczające ilość nieudanych prób.   
- moduły typu account -  odpowiadają za autoryzacje, więc znajdują się tutaj moduły sprawdzające czy użytkownik może się zalogować, 
- moduły typu session odpowiadające za bezpieczeństwo, środowisko, limity, rejestrację sesji, 
- załączone pliki common - na końcu.

## pam_nologin.so
- sprawdza, czy istnieje plik `/etc/nologin`,
- jeśli tak, blokuje logowanie zwykłych użytkowników (nie wylogowuje aktywnych sesji),
- root może się zalogować zawsze, 
- po zakończeniu prac plik usuwa się, system wraca do normalnej pracy 

## Red flags SOC
- usunięcie modułu pam_nologin.so,
- zmiany w nietypowych/niestandardowych modułach PAM (np.: `pam_security.so`),
- manipulacje w typie session (zmiana limitów, grup, SELinux, itd.),
- zmiany w plikach common,
- login działający gdy system powinien być zamknięty (backdoor).

## Wnioski
- dobrą praktyką jest dodawanie plików `common` na końcu, za specyficznymi dla danej aplikacji ustawieniami, 
- ważne jest sprawdzanie kto i kiedy zmienił plik nologin w katalogu `/etc/` za pomocą polecenia stat (`mtime`, `ctime`, `access`).
- Należy sprawdzać czy moduły nologin są poprawnie ustawione w plikach PAM, czy nie zostały usunięte,
- Należy sprawdzać metadane plików common oraz PAM dla aplikacji, a także treść tych plików. 





