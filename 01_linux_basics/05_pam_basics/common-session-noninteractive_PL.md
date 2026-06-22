# /etc/pam.d/common-session-noninteractive

## Cel laboratorium
Zrozumienie do czego służy common-session-noninteractive i jak jest konfigurowany.

## Czym jest /etc/pam.d/common-session-noninteractive
Jest to plik globalny używany przez wszystkie procesy, które nie wymagają pełnej sesji (sudo, cron, skrypty). Jest to lżejsza wersja pliku common-session, która zawiera mniej modułów. Moduły zależą w dużej mierze od rodzaju dystrybucji, w dystrybucjach produkcyjnych może być stosowana większa liczba modułów, w zależności od polityki lub potrzeb audytu.

## Moduły i ich kolejność
- `pam_limits.so` - nakłada limity zasobów CPU, pamięci, max procesów. Niezależnie od dystrybucji powinien występować z flagą `required`. Jego brak może powodować, że DoS lub procesy noninteractive mogą zasypywać system,
- `pam_env.so` - ustawia zmienne środowiskowe (PATH, LANG, etc.). Niezależnie od rodzaju dystrybucji - flaga `required`. Brak modułu sprawia, że proces może mieć nieprawidłowe środowisko,
- `pam_keyinit.so` - inicjalizuje keyrings (przestrzeń kluczy). Flaga `optional` dla systemów edukacyjnych, `required` dla systemów produkcyjnych. Brak keyring może utrudnić audyt lub działanie niektórych procesów, 
- `pam_loginuid.so` - ustawia AUID, dzięki czemu w audycie widać faktycznego użytkownika. Nie występuje w systemach edukacyjnych, w produkcyjnych powinien mieć flagę `required`. Brak modułu -> logi audytu nie pokażą faktycznego użytkownika,  
- `pam_systemd.so` - rejestruje sesję w systemd, start/koniec, UID. W systemach edukacyjnych zwykle występują z flagą `optional`, w produkcyjnych zależy od polityki - `required`. Jeśli moduł ma flagę optional i się nie wykona lub go brak - brak śladu procesu w journald, utrata audytu,
- `pam_umask.so` - ustawia domyślny umask dla procesu. W systemach edukacyjnych występuje z flagą `optional`, w produkcyjnych zależy od polityki `optional/required`. Wpływa tylko na pliki tworzone w trakcie procesu noninteractive. Brak modułu może powodować, że procesy mogą tworzyć pliki z nieprawidłowymi uprawnieniami.
Kolejność modułów wpływa na przygotowanie środowiska procesu - np. `pam_env.so` powinien być wykonany przed `pam_systemd.so`, aby sesja została zarejestrowana z poprawnymi zmiennymi środowiskowymi. 

## Wnioski bezpieczeństwa
- w systemach produkcyjnych warto jest stosować moduły umożliwiające pełny audyt, inaczej potencjalny atakujący może ukryć swoje działania, 
- należy przestrzegać prawidłowej kolejności oraz prawidłowych flag modułów, ponieważ może to powodować niewłaściwe działanie sesji, np. moduł `pam_env.so` za modułem `pam_systemd.so` może powodować dziwne błędy,
- brak modułu limits niesie za sobą ryzyko wzrostu zużycia zasobów (DoS),
- nieprawidłowe ustawienie `pam_env.so` daje atakującemu możliwość nadpisania `PATH`, `LD_LIBRARY_PATH`,
- brak modułów `pam_loginuid.so` i `pam_systemd.so` - to utrata śladów procesów noninteractive,
- moduły `pam_permit.so` lub `custom.so`, które pozwalają na wszystko, to krytyczny błąd, tylko malware lub bardzo zły administrator może to ustawić,
- noninteractive session = krótka sesja dla jednego polecenia, więc brak `systemd/loginuid` -> utrata audytu tylko dla tej sesji, nie dla całego konta,
- system może być poprawnie zabezpieczony pod względem wykonania, ale jednocześnie nieaudytowalny, co stanowi istotne ryzyko z perspektywy SOC.

## Analiza przypadków (Case study)

### Case study 1
```
session optional pam_limits.so  
session required pam_env.so  
session optional pam_systemd.so  
session optional pam_permit.so  
session optional pam_umask.so  
```
Analiza:  
- analiza systemu produkcyjnego,
- moduł `pam_limits.so` - niepoprawne i niebezpieczne ustawienie flagi, może skutkować potencjalnym atakiem DoS, ponieważ jeśli moduł się nie wykona, nie nałoży limitów na sesję. Ten moduł musi się wykonać dla bezpieczeństwa sesji,
- `pam_env.so` posiada właściwą flagę, ustawia zmienne środowiskowe; ich nieprawidłowa konfiguracja może umożliwić manipulację PATH/LD_LIBRARY_PATH,
- `pam_systemd.so` - w systemach produkcyjnych dobrze mieć wgląd do logów procesów noninteractive, w tym przypadku istnieje ryzyko, że gdy moduł się nie wykona, nie będzie mieć wpływu na końcowy wynik typu i może skutkować brakiem informacji o rozpoczęciu i zakończeniu sesji - to utrudnia audyt,
- `pam_permit.so` - moduł udzielający pozwolenia nie powinien się znajdować w pliku common-session-noninteractive, to potencjalny backdoor. Obecność tego modułu powoduje, że logika bezpieczeństwa PAM zostaje unieważniona, ponieważ moduł ten zawsze zwraca sukces,
- brakuje modułu `pam_loginuid.so`, który umożliwiłby pełny audyt, pokazujący, jaki użytkownik wykonał polecenie z sudo (AUID). Ten moduł powinien znajdować się przed modułem `pam_systemd.so`, 
- flaga modułu `pam_umask.so` pozwala na scenariusz, w którym moduł się nie wykona, a więc stwarza ryzyko tworzenia zbyt otwartych plików. Opcjonalnie można rozważyć zmianę flagi na `required`.   

Sugerowane działania:  
- zmiana flagi `pam_limits.so` na `required`,
- zmiana flagi `pam_systemd.so` na `required` (zalecana do ułatwienia audytu, pozostawienie `optional` to świadome ograniczenie audytu),
- usunięcie modułu `pam_permit.so` - to niebezpieczny moduł, który może wskazywać na potencjalny backdoor, 
- sprawdzenie czasu modyfikacji pliku common-session-noninteractive oraz autora zmiany, zweryfikowanie czy zmiana była uprawniona i autoryzowana. `pam_permit.so` + `optional` = zawsze sukces - to jest krytyczne,
- sprawdzenie polityki firmy, czy nie wymaga pełnego audytu, jeśli tak dodać moduł `pam_loginuid.so` z flagą `required` - to zapewni widoczność AUID,
- po wprowadzeniu zmian, kolejność, flagi oraz cała konfiguracja będzie poprawna i bezpieczna dla systemu i jego kontroli.   

### Case study 2
```
session required pam_limits.so   
session required pam_env.so  
session required pam_systemd.so  
session required pam_umask.so  
```
Analiza:  
- konfiguracja zapewnia bezpieczeństwo wykonania, ale nie zapewnia audytowalności,  
- brakuje modułu `pam_loginuid.so`, który powinien występować z flagą `required` przed modułem `pam_systemd.so`. Może występować kilka przyczyn braku tego modułu:
	- ktoś przeoczył, 
	- system działa więc nikt nie zauważył modułu, ponieważ konfiguracja na pierwszy rzut oka wygląda bezpiecznie,
	- ktoś celowo usunął moduł, by móc zostać niewidocznym w trakcie użycia polecenia z sudo. 
	Bez tego modułu logi w journalctl:
	- pokaże proces,
	- pokaże UID (np. root),
	- nie pokaże AUID (oryginalnego użytkownika).  

Sugerowane działanie:
- dodać moduł `pam_loginuid.so` z flagą `required` przed modułem `pam_systemd.so`,
- należy również sprawdzić, czy brak modułu `pam_loginuid.so` był celowy, czy wynika to z przeoczenia - w tym drugim przypadku należy przeprowadzić audyt historycznych logów sudo. 