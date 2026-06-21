# Bity specjalne - SUID, SGID, Sticky bit

## Cel
Zrozumienie jakie bity specjalne występują w systemie Linux i jakie mają zastosowanie.

## SUID (Set User ID) 
Pojawia się jako `s` zamiast `x` w miejscu uprawnień dla właściciela
- działa wyłącznie na plikach wykonywalnych,
- bit ten powoduje, że program uruchomiony przez zwykłego użytkownika działa z uprawnieniami właściciela pliku. Stosuje się go w sytuacjach gdy zachodzi potrzeba by użytkownik mógł wykonać uprzywilejowaną operację,
- przykład: 
	- `/usr/bin/su` ma uprawnienia `-rwsr-xr-x` pozwala użytkownikowi przełączyć się na innego użytkownika,
- zagrożenie: 
	- jeśli program z `suid` ma wadę (podatność źle zarządza zmiennymi środowiskowymi), atakujący może przejąć kontrolę i działać jako root,
- jak ustawić: 
	- `chmod u+s /ścieżka/do/pliku`
- detekcja: 
	- polecenie `find / -perm -4000 2>/dev/null` wyświetli listę wszystkich `SUID` w systemie. Należy monitorować podejrzane pozycje (np. nano, vim, less jeśli mają sudo mogą oznaczać backdoor - nie powinny występować). 

## SGID (Set Group ID) 
Bit ten działa inaczej na plikach a inaczej na katalogach:
- `SGID` na plikach:
	- działanie: 
		- bit ten ma podobne działanie do `SUID` jednak dotyczy grupy a nie właściciela,
	- zagrożenie:
		- podobnie jak `SUID` - jeśli program ma jakąś podatność atakujący może uzyskać uprawnienia grupy (w tym uprawnienia roota jeśli grupa to root),
	- jak ustawić: 
		- `chmod g+s /ścieżka/do/pliku`,
	- detekcja: 
		- polecenie `find / -perm -2000 2>/dev/null` wyświetli listę wszystkich `SGID` w systemie,
- `SGID` na katalogach: 
	- działanie: 
		- katalog ustawiony z `SGID` powoduje, że każdy nowo utworzony plik/katalog wewnątrz tego katalogu dostanie tę samą grupę co katalog, a nie grupę domyślną użytkownika, 
	- zagrożenie: 
		- to luka, należy uważać jakie pliki i katalogi tworzy się w katalogu z `SGID`, by nie znalazł się w grupie plik, który nie powinien się tam znajdować, 
		- `SGID` jest pożyteczny i pożądany w katalogach dotyczących konkretnych projektów, bo to ułatwia współpracę, jeśli jednak `SGID` znajduje się na katalogu `/tmp` to stanowi zagrożenie, ponieważ oznaczałoby to, że pliki tworzone w `/tmp` dziedziczyłyby grupę katalogu co zaburza izolację między użytkownikami,
		- katalogi `SGID` z zapisem dla innych - to duże ryzyko, ponieważ jeśli katalog ma ustawiony `SGID` i każdy użytkownik ma prawo zapisu, to tworzone tam pliki będą należeć do grupy katalogu. W ekstremalnych przypadkach, gdy grupą jest root, zwykły użytkownik mógłby stworzyć plik, który wykona się z uprawnieniami grupy root,
		- `SGID` nie powinien znajdować w katalogach domowych i ich podkatalogach, (`/home`). Jeśli jednak się znajduje, to najprawdopodobniej błąd konfiguracji, prywatność użytkownika jest zagrożona,
		- `SGID` nie powinien się znajdować również w katalogach, w których są instalowane niestandardowe aplikacje (np. `/opt`, `/srv`),
	- jak ustawić: 
		- `chmod g+s /ścieżka/do/katalogu`.
	- detekcja: 
		- `ls -ld katalog` - jeśli w uprawnieniach grupy dla wykonania katalogu znajduje się `s` oznacza, że katalog jest oznaczony `SGID`. Należy szukać podejrzanych `SGID` na katalogach, na których ich być nie powinno oraz sprawdzać uprawnienia dla katalogów z `SGID` czy zapis dla innych jest wyłączony.

## Sticky bit
- stosowany wyłącznie na katalogach (oznaczony jako `t` w prawach wykonywania dla innych (np. drwxrwxrwt), 
- działanie: 
	- powoduje, że tylko właściciel pliku, (lub root) może usunąć lub zmienić nazwę pliku w tym katalogu, nawet jeśli wszyscy mają prawo zapisu, 
	- stosowany dla katalogów `/tmp` i `/var/tmp`,
- zagrożenie: 
	- brak ustawionego `sticky bit` na katalogach `/tmp` i `/var/tmp`, każdy może kasować cudze pliki (DoS), 
- jak ustawić: 
	- `chmod +t /ścieżka/do/katalogu`,
- detekcja: 
	- `ls -ld /tmp /var/tmp`. 

## Wnioski bezpieczeństwa
- należy monitorować katalogi `/tmp` i `/var/tmp` pod kątem sticky bitu, zawsze powinien być tam ustawiony, by zapobiec potencjalnemu zagrożeniu DoS, 
- należy sprawdzać zasadność przydzielania plików SUID i SGID w plikach i katalogach, bity te w niewłaściwych katalogach mogą stwarzać zagrożenie eskalacji uprawnień, backdoor i niebezpieczeństwa wykonania potencjalnych złośliwych skryptów, 
- w programach i skryptach nie należy się opierać na domyślnych bibliotekach, które mogą zostać podmienione przez np. PATH, LD_PRELOAD, tylko podawać pełne ścieżki do programów. 