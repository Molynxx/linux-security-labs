# /etc/pam.d/common-session

## Cel 
Celem tego laboratorium jest zrozumienie co robi i jak działa plik common-session.

## Czym jest /etc/pam.d/common-session
To jest faza sesji. Typ session działa zupełnie inaczej niż auth (uwierzytelnianie) czy accout (autoryzacja). Od typu session zależy wszystko co się dzieje po potwierdzeniach od auth i account. Zadania związane z typem session:
- tworzenie i czyszczenie sesji użytkownika,
- ustawianie zmiennych środowiskowych, 
- ograniczenia zasobów,
- logowanie aktywności. 

## Moduły typu session i ich kolejność 
W tym module występują zupełnie inne moduły (nie licząc pam.unix.so) niż w typach auth i account. Wśród modułów typu session wyróżnić można:
- pam_limits.so - moduł ten nakłada limity na zasoby i procesy, z których może korzystać sesja użytkownika. Jakie limity można tu ustawić:
	- nproc - ile procesów może uruchomić użytkownik jednocześnie,
	- nofile - ile plików może mieć jednocześnie otwartych,
	- core - czy wolno tworzyć core dumpy,
	- fsize - jak duży plik może stworzyć,
	- memlock - czy może zablokować pamięć RAM,
	- cpu - ile czasu CPU może zużyć.
- pam_env.so - odpowiada za inicjalizację bezpiecznego środowiska użytkownika.
	- ustawia zmienne środowiskowe, 
	- usuwa / nadpisuje niebezpieczne wartości,
	- zapewnia, że sesja startuje w przewidywalnym stanie. 
	pam_env czyta pliki /etc/environment oraz /etc/security/pam_env.conf
- pam_umask.so - to jest instrukcja jakie uprawnienia mają być zabrane nowo tworzonym plikom i katalogom. On nie ustala uprawnień, on zabiera część z nich w momencie otwarcia sesji. 
	- domyślnie pobiera wartość umask z konfiguracji systemowej (np.  /etc/login.defs, /etc/profile, /etc/bashrc - zależnie od dystrybucji),
	- opcjonalnie można wymusić konkretną wartość bezpośrednio w linii modułu PAM, np.: session optional pam_umask.so umask=027.
- pam_unix.so - odpowiada za otworzenie, utrzymanie i zamknięcie sesji użytkownika. Rejestruje w systemie:
	- /var/run/utmp - aktywne sesje,
	- /var/log/wtmp - historia logowań, 
	- /var/log/lastlog - dzięki czemu polecenia who, w, last mogą dać informacje na temat użytkowników online lub kiedy się logowali.
	Moduł ten korzysta z danych już zweryfikowanych w auth i respektuje decyzje z account. 
pam_systemd.so - rejestruje sesję użytkownika w systemd - logind i podpina ją pod mechanizmy zarządzania procesami systemd. Tu należy odróżnić, ponieważ unix również prowadzi proces rejestracji jednak te dwa moduły różnią się zapisem w plikach. Unix zapisuje w /var/run/utmp, /var/log/wtmp oraz w /var/log/lastlog, a systemd zapisuje w pliku logind. Dzięki niemu każda sesja ma swój ID. Zadania modułu systemd:
	- rejestruje sesję w logind
		- sesja ma ID,
		- wiadomo kto, jak i kiedy.
	- tworzy scope dla sesji - session-xx.scope,
	- podpina procesy pod cgroups - wszystkie procesy sesji = jeden koszyk, 
	- umożliwia cleanup przy logout - logout = kill wszystkiego w scope,
	- umożliwia korelację zdarzeń. 
	To GŁÓWNY moduł, który integruje PAM z systemd-logind i dzięki temu widzimy sesję w loginctl. 
- pam_loginuid.so - przypisuje UID (AUID) użytkownika do wszystkich procesów w sesji. Dzięki temu istnieje możliwiśc sprawdzenia kto otworzył proces nawet jeśli otworzył sesję z podwyższonymi uprawnieniami. AUID (Audit UID) identyfikuje pierwotnego użytkownika, który rozpoczął sesję i nie zmienia się przy eskalacji uprawnień (np. sudo, su).
W przeciwieństwie do UID /EUID, które mogą być używane zamiennie w zależności od bieżących uprawnień procesu, AUID pozwala systemowi i mechanizmom audytu jednoznacznie ustalić, kto faktycznie wykonał daną akcję, nawet jeśli działał jako root.
- pam_exec.so - to moduł pozwalający uruchomić zewnętrzny program lub skrypt na określonym etapie PAM. Jeżeli moduł ma za zadanie uruchomić coś w czasie otwierania sesji, to należy pamiętać, że sesja może się otwierać wiele razy, bo otwiera się za każdym razem gdy następuje logowanie, przejście na innego użytkownika, użycie polecenia podwyższającego uprawnienia. PAM przed uruchomieniem skryptu ustawia zmienne środowiskowe.
- pam_motd.so - to prosty moduł, który ma tylko jedno zadanie - wyświetlić Message Of The Day użytkownikowi po otwarciu sesji. Pliki z komunikatem znajdują się w:
	- /etc/motd - statyczny tekst, 
	- /run/motd.dynamic - dynamiczny, np.: informacje o aktualizacjach. 
oba te źródła można podać w opcjach modułu, jednak jeśli opcja pozostaje pusta, to wtedy moduł użyje domyślnego pliki i sprawdzi w kolejności lokalizacje: 
	- /run/motd.dynamic - jeśli nic tu nie znajdzie sprawdza dalej,
	- /etc/motd.
	Jeżeli nigdzie nie znajdzie pliku po prostu nie wyświetla nic. 
- pam_mail.so - ten moduł działa bardzo podobnie do motd. Zadaniem modułu mail jest sprawdzenie czy istnieje plik poczty w /var/mail/username lub /var/spool/mail/username i czy nie jest pusty. Jeśli plik istnieje i nie jest pusty, moduł wyświetli użytkownikowi powiadomienie o nowej poczcie po otwarciu sesji. Nie zawsze stosowany w nowoczesnych dystrybucjach ale warto wiedzieć ze taki istnieje. 
- pam_lastlog.so - moduł zapisuje i aktualizuje w pliku /var/log/lastlog informacje o ostatnim logowaniu użytkownika. Zapis dotyczy każdej otwieranej sesji. Wyświetlić zapisane dane można za pomocą polecenia last lub lastlog. Moduł ten można użyć z opcją 'showfailed', która wyświetla użytkownikowi po otworzeniu sesji informację na temat nieudanych prób logowania. 

## Flagi kontrolne modułów
- pam_limits.so - 'required', bo limity muszą zostać nałożone,
- pam_env.so - 'required', ponieważ poprawne ustawienie środowiska jest krytyczne dla dalszego działania sesji, a jego niepowodzenie nie powinno być ignorowane. 
- pam_umask.so - 'optional', ponieważ:
	- brak ustawienia umask nie blokuje sesji,
	- system i tak ma: 
		- domyślną umask kernela, 
		- ustawienia w /etc/login.defs, /etc/profile.
	Więc sesja technicznie może działać poprawnie nawet bez pam_umask. 
- pam_unix.so - 'required', bo sesja miso się wykonać w całości inaczej:
	- nie wykona się przy open -> brak rejestracji sesji,
	- nie wykona się przy close -> brak cleanup.
- pam_systemd.so - 'required'. Awaria modułu nie powinna przerwać cleanupu pozostałych mechanizmów. Brak wykonania pam_systemd powoduje, że sesja nie zostaje zarejestrowana w systemd-logind, nie są tworzone scope ani cgroups, przez co systemd traci możliwość poprawnego zarządzania i sprzątania procesów należących do sesji. 
- pam_loginuid.so - 'required', ponieważ bez ustawionego loginuid mechanizmy audytu nie są w stanie jednoznacznie powiązać działań procesów z pierwotnym użytkownikiem sesji.
- pam_exec.so - 'optional', ponieważ moduł ten służy do uruchamiania zewnętrznych skryptów pomocniczych, których niepowodzenie nie powinno blokować sesji. Jednak jego konfiguracja wymaga szczególnej uwagi z punktu widzenia bezpieczeństwa.
- pam_mail.so - 'optional' - to moduł informacyjny, nie wpływa na bezpieczeństwo sesji ani na audyt. 
- pam_motd.so - 'optional', z dokładnie tych samych powodów co moduł pam_mail.so.
- pam_lastlog.so - 'optional'. Jest do moduł pomocniczy, informacyjny, bez niego nie ma zagrożenia utrudnienia audytu, są od tego inne zabezpieczenia. 

## Zagrożenia PAM common-session
Faza session w PAM odpowiada za szereg powiązanych zdarzeń:
- otwarcie i zamknięcie sesji użytkownika, 
- poprawne zainicjalizowanie kontekstu wykonawczego procesów, 
- rejestrację sesji, 
- zapewnienie widoczności i możliwości pełnego audytowania działań użytkownika. 
W związku z tym zagrożenia w tej fazie nie mają charakteru binarnego, a ich skutki często ujawniają się pośrednio, w czasie lub dopiero na etapie analizy incydentu.

### Utrata spójności sesji i sprzątania procesów
Obejmuje:
- niepełne wykonanie sesji, 
- brak prawidłowego zamknięcia sesji, 
- niejednoznaczny stan systemu po zakończeniu sesji.
Skutki:
- procesy pozostające po zakończeniu sesji, 
- zasoby niezwolnione po wylogowaniu,
- nieprzewidywalne zachowanie systemu.
Przykładowe moduły: pam_unix.so, pam_systemd.so.

### Utrata audytu i tożsamości użytkownika
Obejmuje:
- brak jednoznacznego przypisania procesów do użytkownika,
- utratę ciągłości śladu audytowego,
- aktywność użytkownika niewidoczna w logach sesyjnych.
Skutki:
- utrudniona analiza incydentów, 
- brak korelacji zdarzeń,
- problemy ID i forensics
Przykładowe moduły: pam_unix.so, pam_loginuid.so.

### Manipulacja środowiskiem wykonawczym i politykami
Obejmuje:
- modyfikację zmiennych środowiskowych, 
- obejście polityk bezpieczeństwa, 
- wykonanie nieautoryzowanego kodu przy starcie lub zamknięciu sesji.
Skutki:
- eskalacja uprawnień (pośrednia),
- persistence,
- nieprzewidywalne zachowanie aplikacji.
Przykładowe moduły: pam_env.so, pam_exec.so, pam_limits.so, pam_umask.so.
Ta kategoria obejmuje całą, szeroką klasę zagrożeń wynikających z wpływu na kontekst uruchomieniowy procesów i absolutnie nie powinna być traktowana jako pojedynczy wektor ataku.

### Problemy zarządcze i obserwowalności systemu
Obejmuje: 
- brak wiarygodnych informacji o sesjach, 
- błędne dane dla narzędzi administracyjnych, 
- utrudnione monitorowanie aktywnych użytkowników.
Skutki:
- błędne decyzje administracyjne, 
- opóźniona reakcja na zagrożenia, 
- obniżona skuteczność detekcji.
Przykładowe moduły: pam_lastlog.so, opcjonalnie pam_motd.so, pam_mail.so

### Ryzyko socjotechniki / manipulacji użytkownikiem
Obejmuje:
- wiadomości powitalne,
- motd,
- maile wysyłane przy logowaniu. 
Skutek:
- użytkownik może zostać nakłoniony do uruchomienia złośliwego kodu, kliknięcia linku lub podania danych. 
Przykładowe moduły: pam_motd.so, pam_mail.so.
Choć mechanizmy te nie wpływają bezpośrednio na bezpieczeństwo techniczne, stanowią istotny wektor zagrożeń pośrednich, ponieważ najsłabszym ogniwem jest zawsze człowiek, szczególnie w kontekście socjotechniki i insider threat.

## Wnioski 
- W tym typie (session) musi zostać wykonanych wiele czynności, aby system był bezpieczny. Pierwszym krokiem jest sprawdzenie, czy wszystkie wymagane moduły sesji istnieją w stosie oraz czy mają poprawnie ustawione flagi. W przeciwnym razie może dojść do przerwania stosu lub pominięcia istotnych modułów, co zagraża bezpieczeństwu systemu. 
- Kolejność modułów w common-session jest krytyczna: najpierw inicjalizacja środowiska (env, limits, umask), następnie rejestracja sesji (unix) oraz integracja z systemd, który tworzy scope i cgroups sesji. Zaraz po tym następuje zapis identyfikatora użytkownika dla audytu. Zmiana kolejności może skutkować nieprawidłowym przypisaniem procesów, utratą audytu i problemami ze sprzątaniem po sesji. Pozostałe moduły, które nie wpływają bezpośrednio na te zdarzenia, wykonywane są dopiero później.
- Wszystkie moduły, które mają w opcjach ustawioną ścieżkę (exec, motd, mail) powinny być szczegółowo sprawdzane i monitorowane, ze względu na ryzyko podmiany ścieżki, w wyniku czego może dojść do uruchomienia złośliwego skryptu (exec) lub ataku socjotechnicznego (motd, mail). W przypadki istnienia tych modułów w PAM, należy sprawdzić ścieżkę, kto jest właścicielem pliku, jakie są prawa (zapis) do tego pliku/katalogu i czy jest monitorowany przez File Integrity Monitoring (FIM).

## Przykłady analizy konfiguracji pliku common-session (case study)

### Przypadek 1

session required pam_env.so  
session required pam_limits.so  
session optional pam_umask.so  
session required pam_unix.so  
session required pam_systemd.so  
session required pam_loginuid.so  
session optional pam_motd.so  
session optional pam_mail.so  
session optional pam_exec.so
session optional pam_lastlog.so  

Analiza:  
To jest przykład prawidłowego ustawienie common-session. Moduły są w odpowiedniej kolejności a ich flagi są poprawne, model sprawia, że nasza sesja jest bezpieczna. Jedyne co zawsze wymaga sprawdzenia to linki do modułu exec, motd i mail oraz czy do plików znajdujących się w źródle nie zostały utworzone symlinki.   
Sugerowane działanie:  
Nie wymagane 

### Przypadek 2 

session required pam_unix.so  
session optional pam_umask.so  
session required pam_env.so  
session required pam_limits.so  
session required pam_systemd.so  
session required pam_loginuid.so  
session optional pam_motd.so  
session optional pam_mail.so  
session optional pam_exec.so
session required pam_lastlog.so 

Analiza:   
Logicznie są tutaj niewielkie błędy. Choć kolejność modułów choć nie jest zgodna z polityką najpierw środowisko, potem rejestracja to jednak nie stanowią istotnego zagrożenia dla bezpieczeństwa sesji ani systemu. Moduł pam_unix.so z punktu widzenia logiki powinien znajdować się po modelu limits a przed systemd, ale obecne ustawienie nie budzi braki zaufania. Jedyny błąd w tym modelu to flaga 'required' przy module lastlog. To jest moduł pomocniczy, który nie musi się wykonać a jednocześnie nie powinien zakłócać wykonania sesji.  
Sugerowanie działanie:   
Zmienić flagę moduły lastlog na optional. 

### Przypadek 3 

session required pam_env.so  
session required pam_limits.so   
session required pam_unix.so 
session optional pam_umask.so   
session optional pam_systemd.so  
session required pam_loginuid.so  
session optional pam_motd.so  
session optional pam_mail.so  
session optional pam_exec.so
session optional pam_lastlog.so 

Analiza:  
Kolejność modułów w tym przypadku nie budzi podejrzeń, choć umask mogłoby znajdować się przed unix. Krytycznym jednak problemem w tym przypadku jest flaga modułu pam_systemd.so. Jeśli ten moduł się nie wykona, to sesja będzie się wykonywać nadal i jednocześnie nie będzie zarejestrowana w pliku logind, mechanizmy zarządzania nie będą zintegrowane z systemd, nie zostanie utworzone scope ani cgroups. Dzięki scope i cgroups możliwe jest odpowiednie zarządzanie procesami, ponieważ wszystkie procesy użytkownika są podpięte do cgroups, więc wiadomo, które należy zamknąć gdy użytkownik kończy sesję. To dzięki temu modułowi może się wykonać poprawny cleanup. Jeśli ten moduł się nie wykona, procesy zostają po wylogowaniu, malware może łatwiej przetrwać a IR nie wie jakie procesy może zamknąć bez ryzyka zniszczenia dowodów. To bardzo istotny problem z punktu widzenia bezpieczeństwa.   
Sugerowane działanie:  
Zmienić flagę modułu systemd na 'required', bo moduł ten musi się wykonać. 

### Przypadek 4

session required pam_env.so  
session required pam_limits.so  
session optional pam_umask.so   
session required pam_systemd.so  
session required pam_loginuid.so  
session optional pam_motd.so  
session optional pam_mail.so  
session optional pam_exec.so
session optional pam_lastlog.so  

Analiza:  
Kolejny przykład niewłaściwego ustawienia PAM, brak krytycznego modułu - pam_unix.so. Taka konfiguracja może 'oślepić' SOC/IR. Bez tego modułu nie ma wpisów w /var/run/utmp, /var/log/wtmp, /var/log/lastlog. To sprawia, że choć procesy użytkownika są widoczne, nie można sprawdzić czy użytkownik jest online (who, w), kiedy i jak się zalogował i czy się wylogował. Sesja jest niewidoczna. Utrudnia to korelację zdarzeń a sesja może nigdy nie zamknąć się logicznie. TO utrata kontroli i obserwalności.  
Sugerowane działanie:  
Dodanie modułu pam_unix,so z flagą 'required' przed modułem pam_systemd.so.

### Przypadek 5

session required pam_env.so  
session optional pam_limits.so  
session optional pam_umask.so  
session required pam_unix.so  
session required pam_systemd.so   
session optional pam_exec.so  
session optional pam_motd.so  
session optional pam_mail.so
session optional pam_lastlog.so  

Analiza:  
W tym modelu występują dwa istotne dla bezpieczeństwa błędy. Moduł pam_limits.so ma flagę optional, a to oznacza, że moduł może się nie wykonać, a jeśli tak się stanie, nie zostaną nałożone na sesję żadne limity. To może prowadzić do fork bomb (zabicia systemu przez użytkownika tysiącami procesów), lokalnego DoS (zapchanie RAM, FD, CPU) lub core dumpów (zrzuty wrażliwej pamięci). Może także utrudniać pracę IR, ponieważ na jednego użytkownika może przypadać setki tysięcy procesów. Brak wykonania modułu limits to brak kontroli nad szkodami jakie może wyrządzić użytkownik. Drugim poważnym problemem jest brak modułu pam_logiuid.so. To moduł, który zapisuje UID użytkownika (AUID). Podczas gdy UID kernela zależy od uprawnień procesów, AUID nadal pamięta i wskazuje na użytkownika aż do zakończenia sesji. Dzięki temu można sprawdzić jaki użytkownik jest odpowiedzialny za proces. Bez niego wystąpiłyby luki w logach, np.: w momentach gdy użytkownik wykona polecenie z podwyższonymi uprawnieniami, to proces jest widoczny jako proces roota, zamiast użytkownika, który wywołał proces z uprawnieniami roota. Dzięki temu modułowi można rozróżnić root z crona i root z loginu użytkownika, zobaczyć eskalację uprawnień, korelować logowanie, proces, syscall, incydent. To bardzo ułatwia audyt.   
Sugerowane działanie:   
Zmienić flagę modułu pam_limits.so na 'required', dodać moduł pam_logiuid.co za modułem pam_systemd.so

### Przypadek 6

session optional pam_env.so  
session required pam_limits.so  
session optional pam_umask.so  
session required pam_unix.so  
session required pam_systemd.so  
session required pam_loginuid.so  
session optional pam_motd.so  
session optional pam_mail.so  
session optional pam_exec.so
session optional pam_lastlog.so  

Analiza:   
W tym przykładzie występuje jeden ale bardzo poważny błąd. Moduł pam_env.so jest oznaczony flagą 'optional', a to oznacza, że moduł ten może się nie wykonać. Zagrożenia wynikające z niewykonania się tego modułu to bardzo szeroka klasa zagrożeń manipulacji zmiennymi środowiskowymi. Atakujący może przechwytywać dane, podmieniać funkcje, logować dane w tle, niszczyć dane, tworzyć nowych użytkowników, edytować sudoers, podmieniać biblioteki. Może spowodować uruchomienie dowolnego złośliwego skryptu za pomocą modyfikacji zmiennych PATH, TMPDIR lub też uszkodzić system, wywołać niewłaściwe polecenie, zmienić argumenty za pomocą zmiennych środowiskowych LOCALE i IFS. Dobrą praktyką jest używać w skryptach zmiennej zawsze w cudzysłowie, co niweluje błędy wynikające z modyfikacji LOCALE i IFS. Podsumowując brak przewidywalnego środowiska otwiera atakującemu drogę do wykonanie wielu rodzajów ataków.  
Sugerowane działanie:   
zmienić flagę modułu pam_env.so na 'required'.

