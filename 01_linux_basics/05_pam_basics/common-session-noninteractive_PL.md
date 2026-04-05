# /etc/pam.d/common-session-noninteractive

## Cel laboratorium
Zrozumienie do czego służy common-session-noninteractive i jak jest konfigurowany.

## Czym jest /etc/pam.d/common-session-noninteractive
Jest to plik globalny używany przez wszystkie procesy, które nie wymagają pełnej sesji (sudo, cron, skrypty). Jest to lżejsza wersja pliku common-session, która zawiera mniej modułów. 

## Moduły i ich kolejność
- `pam_limits.so` - nakłada limity zasobów CPU, pamięci, max procesów. Flaga - `required`, 
- `pam_env.so` - ustawia zmienne środowiskowe (PATH, LANG, etc.). Flaga - `required`, 
- `pam_keyinit.so` - inicjalizuje keyrings (przestrzeń kluczy). Flaga - `required` dla systemów produkcyjnych, 
- `pam_loginuid.so` - ustawia AUID, dzięki czemu w audycie widać faktycznego użytkownika. Flaga -  `required`,
-`pam_systemd.so` - rejestruje sesję w systemd, start/koniec, UID. Flaga - `required`. 
-`pam_umask.so` - ustawia domyślny umask dla procesu. Flaga - `optional/required`. Wpływa tylko na pliki tworzone w trakcie procesu noninteractive. 
Kolejność modułów wpływa na przygotowanie środowiska procesu.

## Wnioski bezpieczeństwa
- w systemach produkcyjnych warto jest stosować moduły umożliwiające pełny audyt, inaczej potencjalny atakujący może ukryć swoje działania, 
- należy przestrzegać prawidłowej kolejności oraz prawidłowych flag modułów, ponieważ może to powodować niewłaściwe działanie sesji, 
- brak modułu limits niesie za sobą ryzyko wzrostu zużycia zasobów (DoS),
- nieprawidłowe ustawienie `pam_env.so` daje atakującemu możliwość nadpisania PATH, LD_LIBRARY_PATH,
- brak modułów `pam_loginuid.so` i `pam_systemd.so` - to utrata śladów procesów noninteractive,
- moduły `pam_permit.so` lub `custom.so`, które pozwalają na wszystko to krytyczny błąd.
- noninteractive session = krótka sesja dla jednego polecenia, więc brak `systemd/loginuid` -> utrata audytu tylko dla tej sesji, nie dla całego konta,
- system może być poprawnie zabezpieczony pod względem wykonania, ale jednocześnie nieaudytowalny, co stanowi istotne ryzyko z perspektywy SOC.

## Analiza przypadków (Case study)

### Case study 1

session optional pam_limits.so   
session required pam_env.so   
session optional pam_systemd.so   
session optional pam_permit.so   
session optional pam_umask.so   

Analiza:   
- analiza systemu produkcyjnego,
- moduł `pam_limits.so` - niepoprawna flaga, ryzyko DoS,
- `pam_env.so` ustawienie prawidłowe,
- `pam_systemd.so` - nieprawidłowa flaga, może to utrudnia audyt,
- `pam_permit.so` - moduł udzielający pozwolenia nie powinien się znajdować w pliku common-session-noninteractive, to potencjalny backdoor, zawsze zwraca sukces,
- brakuje modułu `pam_loginuid.so`, brak pełnego audytu (AUID),
- flaga modułu `pam_umask.so` - ryzyko tworzenia zbyt otwartych plików.  

Sugerowane działania:  
- zmiana flagi `pam_limits.so` na `required`,
- zmiana flagi `pam_systemd.so` na required (zalecana do ułatwienia audytu, pozostawienie `optional`to świadome ograniczenie audytu),  
- usunięcie modułu `pam_permit.so` - to niebezpieczny moduł, który może wskazywać na potencjalny backdoor, 
- sprawdzenie czasu modyfikacji pliku common-session-noninteractive oraz autora zmiany, zweryfikowanie czy zmiana była uprawniona i autoryzowana. `pam_permit.so` + `optional` = zawsze sukces - to jest krytyczne,
- sprawdzenie polityki firmy, czy nie wymaga pełnego audytu, jeśli tak dodać moduł `pam_loginuid.so` z flagą `required` - to zapewni widoczność AUID,
- po wprowadzeniu zmian, kolejność, flagi oraz cała konfiguracja będzie poprawna i bezpieczna dla systemu i jego kontroli.   

### Case study 2

session required pam_limits.so   
session required pam_env.so  
session required pam_systemd.so  
session required pam_umask.so  

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
- dodać moduł `pam_loginuid.so` z flagą `required` przed modułem `pam_systemd.so`.