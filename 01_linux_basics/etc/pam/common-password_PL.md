# /etc/pam.d/common-password

## Cel laboratorium
Celem tego laboratorium było zaznajomienie się z rolą jaką pełni typ password w systemie oraz jakie występują w nim moduły i jakie są z nim związane zagrożenia. 

## Czym jest /etc/pam.d/common-password
Typ password jest odpowiedzialny za zmianę hasła w systemie, ale jego rola nie ogranicza się do tej jednej funkcji. Tutaj występuje cały ciąg zdarzeń:
- ustawienie nowego hasła,
- zmiana hasła,
- zapis (hash hasła),
- polityki rotacji,
- reakcje na błędy.
Ten typ nie wykonuje się przy uwierzytelnianiu, autoryzacji ani sesji, on wykonuje się wyłącznie wtedy, kiedy następuje ustawienie lub zmiana hasła. Od polityk oraz modułów tego typu zależy bezpieczeństwo hasła (jego jakość, długość, złożoność, a także niepowtarzalność).

## Moduły typu password i ich kolejność
- pam_pwquality.so - to moduł, który odpowiada za jakość hasła. Po otrzymaniu hasła analizuje je, sprawdza jego jakość i może odrzucić hasło jeśli nie spełnia warunków. W tym module można skonfigurować wiele opcji zapewniających odpowiednią jakość hasła, w tym:
	- minlen - minimalna długość hasła, 
	- dcredit - wymagana ilość cyfr w haśle (wartości ujemne oznaczają minimalną liczbę znaków klasy),
	- ucredit - wymagana ilość wielkich liter w haśle (wartości ujemne oznaczają minimalną liczbę znaków klasy),
	- lcredit - wymagana ilość małych liter w haśle (wartości ujemne oznaczają minimalną liczbę znaków klasy),
	- ocredit - wymagana ilość znaków specjalnych w haśle (wartości ujemne oznaczają minimalną liczbę znaków klasy),
	- difok=N - N oznacza ile znaków musi się różnić od poprzedniego hasła,
	- maxrepeat - maksymalna ilość powtórzeń jednego znaku w haśle,
	- maxclassrepeat - maksymalna liczba powtórzeń tej samej klasy znaków,
	- gecoscheck - sprawdza imię, login i dane z /etc/passwd i nie pozwala używać danych w haśle,
	- dictcheck - sprawdza hasło według słownika, chroni na przykład przed 'Password123',
	- retry - ile razy użytkownik może spróbować wprowadzić nowe hasło,
	- enforce_for_root - domyślnie root nie podlega polityce jakości hasła, dzięki tej opcji można włączyć te polityki także dla roota (nie działa w starszych wersjach PAM i nie zawsze jest respektowana),
	- use_authtok - zapewnia, dobrą współpracę w typie password pracują na tym samym haśle (nowym haśle wprowadzonym przez użytkownika), zapobiegając niespójnościom i obejściom polityki. Ważne: jeśli moduł pwquality jest pierwszym modułem typu - nie używa się w nim tej opcji. Pierwszy moduł pobiera dopiero hasło od użytkownika, więc moduł, który jest pierwszy nie powinien mieć włączonej tej opcji.  
- pam_pwhistory.so - to moduł zapobiegający ponownemu użyciu poprzednich haseł, podobnie jak w powyższym module, moduł ten posiada kilka opcji. Jednak opcje te używane są zwykle inline, ponieważ moduł nie w każdej dystrybucji ma swój plik pwhistory.conf. Wśród opcji modułu wyróżnia się:
	- remember=N - moduł pamięta N ostatnich haseł. które są zapisane w postaci hashy w pliku /etc/security/opasswd,
	- use_authtok - używa tego samego hasła, które już sprawdził pam_pwquality.so i które zapisze pam_unix.so,
	- enforce_for_root - działa analogicznie jak w pam_pwquality.so, z tym, że dotyczy tutaj wymuszenia polityki haseł związaną z historią haseł. 
	- retry=N - ile razy użytkownik może próbować ustawić nowe hasło. 
- pam_sss.so -  moduł ten ma za zadanie zapisania nowego hasła w zewnętrznej bazie (dotyczy konta domenowego). Wśród opcji tego modułu wyróżnia się:
	- use_authtok - używa tego samego hasła które pobrał pierwszy moduł,
	- try_first_pass - używane czasem zamiennie z use_authtok, nakazuje użycie tokenu ustawionego wcześniej,
	- nullock - pozwala na puste hasło, jeśli polityka systemu to dopuszcza, co co w praktyce jest rzadkie i bardzo niezalecane. 
- pam_unix.so - to moduł, który zapisuje nowe hasło do /etc/shadow w postaci hasha. Posiada funkcje wyłącznie inline, a wśród nich wyróżnia się:
	- use_authtok - użyj hasła, które zostało sprawdzone wcześniej przez poprzednie moduły,
	- yescrypt/SHA512 - to opcje określające algorytm hashowania. W nowoczesnych systemach zaleca się yascrypt, natomiast SHA512 jest starszy choć wciąż akceptowany. 
	- shadow - praktycznie zawsze domyślna, mówi zapisuj hasło w /etc/shadow  a nie w /etc/passwd,
	- nullok - niezalecane, używane w systemach embedded, kontach technicznych. Stosowane rzadko, niedopuszczalne w systemach produkcyjnych, niezalecane.
	- remember=N - to stara metoda, obecnie niezalecana - duplikuje funkcję pam_pwhistory.so.
- pam_gnome_keyring.so - synchronizuje hasło systemowe z keyringiem użytkownika, nie wpływa na bezpieczeństwo systemowe (wyłącznie dla kont lokalnych, nie stosuje się w systemach serwerowych ani kontach domenowych).

## Flagi kontrolne modułów
- pam_pwquality.so - 'requisite' jest zalecaną flagą dla tego modułu jeśli celem jest natychmiastowe odrzucenie słabego hasła, 'required' w przypadkach kiedy celem jest kompletność logów.
- pam_pwhistory.so - 'required'. ponieważ ten moduł tylko sprawdza czy hasło było używane wcześniej, reszta modułów powinna się wykonać, by zachować spójność komunikatów i poprawne obsłużyć use_authtok. Flaga 'required' w tym module wspomaga audyt. Jeśli moduł ten zwróci FAIL reszta modułów się wykona, jednak hasło się nie zapisze do pliku shadow, a cały stos zwróci FAIL. 
- pam_sss.so - 'sufficient', ponieważ kiedy zmiana hasła dla konta domenowego zakończy się sukcesem, nie ma już potrzeby wykonania modułu pam_unix.so, który działa dla konta lokalnego, a jednocześnie brak konta domenowego nie zatrzyma stosu. 
- pam_unix.so - 'required' jest najczęściej stosowaną i najbardziej bezpieczną flagą, pozwalającą by system przyszedł przez kolejne moduły w stacku, co ułatwia audyt. Flaga sufficient jest tutaj dopuszczalna przy założeniu, że pam_unix.so jest ostatni w stosie a kolejność modułów jest prawidłowa i logiczna. 
- pam_gnome_keyring.so - 'optional', powodzenie lub porażka tego modułu nie wpłynie na bezpieczeństwo, a jednocześnie nie przerwie stosu w przypadki gdy ten moduł się nie powiedzie, 

## Zagrożenia common-password

### Zagrożenia wynikające z konfiguracji modułów
Każdy moduł w tym typie spełnia inną specyficzną rolę:
- pam_pwquality.so tutaj znajduje się wszystko co dotyczy jakości hasła, brak konfiguracji minlen, dcredit, ucredit, lcredit, ocredit, dictcheck, gecoscheck, enforce_for_root może prowadzić do słabych haseł, podatności na brute force, credential stuffing. 
- pam_pwhistory.so - główne zagrożenia wynikające z tego modułu to brak pamięci haseł, zgoda na powtarzanie tego samego hasła w krótkich odstępach czasu, ponieważ moduł ten nie sprawdza czy hasła nie wyciekły, a takie hasło może stać się backdoorem. Ryzyko backdoora zachodzi także przy braku pliki opasswd, braki modułu w ogóle. Moduł nie chroni przed hasłami, które wyciekły z innych źródeł, dlatego dobrze mieć dodatkowe narzędzia monitorujące wycieki haseł.
- pam_sss.so - błędy związane z tym modułem powodują głownie problemy z integracją. Brak tego modułu spowoduje brak zapisu hasła w zewnętrznej bazie (zmiana hasła nie zadziała). 
- pam_unix.so - podobnie jak poprzedni moduł z tym, że tu następuje zapis do pliku shadow. Należy także pamiętać, że zbyt słaby algorytm hashujący może sprawić, że hashe będą bardziej podatne na złamanie.
- Pam_gnome_keyring.so -nie stwarza żadnych zagrożeń z punktu widzenia bezpieczeństwa.

### Zagrożenia bezpieczeństwa
- błędne flagi kontrolne modułów mogą powodować ominięcie polityki haseł lub w ogóle pominąć dalszą część stosu,
- brak use_authtok powoduje, że moduły mogą otrzymać różne wersje hasła, a to prowadzi do niespójności pozwalających na obejście kontroli jakości. 

## Wnioski bezpieczeństwa
- należy dokładnie sprawdzać czy wszystkie krytyczne dla bezpieczeństwa moduły znajdują się w stosie oraz czy ich kolejność i flagi kontrolne są poprawne. Źle ustawione mogą prowadzić do omijania polityki haseł, uniemożliwić zmianę hasła lub pozostawienia backdoora. 
-należy weryfikować, czy moduły mają ustawioną opcję use_authtok. Pamiętać należy, że pierwszy moduł w stosie nie powinien mieć ustawionej tej opcji. I tu należy też zwrócić uwagę, czy w plikach PAM dla poszczególnych aplikacji nie ma innych wpisów w typie password przed dołączonym plikiem common-password. Dobrą praktyką jest umieszczenie wszystkich potrzebnych modułów w common-password i załączenie go do plików PAM aplikacji.
- należy również uważnie sprawdzać czy opcje wymagane do bezpieczeństwa hasła są dobrze skonfigurowane. W przypadku modułu pam_pwquality.so warto sprawdzić nie tylko opcje wpisane inline, ale także, czy istnieje plik pwquality.conf i czy jest poprawnie skonfigurowany. Pamiętać należy, że opcje inline zawsze nadpisują te znajdujące się w pliku .conf. Zbyt krótkie hasła, retry ustawione na 0, brak dictcheck i gecoscheck mogą powodować zagrożenie z powodu zbyt słabego hasła.   
	### Dodatkowe uwagi dotyczące bezpieczeństwa haseł
	Warto wiedzieć, co ma największy wpływ na bezpieczeństwo haseł. Wbrew pozorom, ustawienie klas hasła nie ma tak istotnego znaczenia jak długość hasła czy sprawdzenie go w słownikach czy danych w pliku /etc/passwd. Patrząc realnie na obecne moce obliczeniowe, nawet hasło określonych klas znaków lecz wystarczająco długie, zawierające nieoczywiste połączenia słów, lub też ciągi znaków nie oznaczające słów są bardzo trudne do odczytania z hasha. Na przykład, hasło mające 16 znaków (bez określonych klas) jest znacznie silniejsze niż hasło mające 8 znaków i określone różne klasy znaków. To zawsze warto brać pod uwagę ustalając politykę haseł w systemie. 
- w kwestii powtarzalności haseł, czyli czynności lubianej przez użytkowników, zabezpieczeniem jest moduł pam_pwhistory.so, który zapisuje N ostatnich haseł w pliku opasswd. W tym module należy zawsze sprawdzać , czy plik /etc/security/opasswd istnieje, czy posiada odpowiednie prawa dostępu (600), czy nie został wyczyszczony, oraz jaka wartość jest ustawiona w opcji remember. Zbyt niska wartość może powodować zbyt często stosowanie tego samego hasła i tylko kilka wymiennie stosowanych haseł. Z drugiej strony nie ma sensu ustawienie tej wartości na 100. Wszystko należy skonfigurować mając na uwadze ustalone okresy ważności haseł. Na przykład, jeśli ważność hasła wynosi 30 dni, to ustawienie wartości remember=24 uniemożliwi powtórzenie hasła w trakcie kolejnych 2 lat. Większe wartości nie dają większego sensu, bo hasła w postaci hashy będą powodować niepotrzebne "rośnięcie" pliku opasswd.
- zawsze należy również sprawdzać jaki algorytm hashowania jest ustawiony w module pam_unix.so. Zbyt słaby algorytm powoduje zagrożenie, że hasło może zostać 'odczytane' z hasha. Warto też sprawdzić czy istnieje plik shadow.

## Przykłady analizy konfiguracji pliku common-password (case study)

### Przypadek 1

password required pam_pwquality.so retry=3  
password required pam_pwhistory.so remember=5 use_authtok  
password sufficient pam_sss.so use_authtok  
password required pam_unix.so use_authtok  

Analiza:  
W tym przypadku można by użyć flagi 'requisite' dla modułu pam_pwquiality.so, dzięki temu gdy hasło jest słabe, nie ma sensu kontynuować. Jednak zdarzają się sytuacje, kiedy flaga 'required' jest użyta celowo, by cały proces było logowany i SOC/IR miał łatwiejszy audyt. Flagi kontrolne i parametry w tym przykładzie są użyte poprawnie i są logiczne. Warto jednak sprawdzić plik pwquality.conf, żeby upewnić się, że pozostałe parametry zapewniające bezpieczne hasło są tam skonfigurowane. Poprawna jest także kolejność modułów, Należy pamiętać, że ten typ działa tylko w przypadku zmiany hasła, a hasło zmieniane jest w zależności od tego, na którym koncie użytkownik jest zalogowany; na domenowym czy lokalnym. W przypadku zmiany hasła na koncie domenowym, moduł odpowiedzialny za zapisane hasła to pam_sss.so. Jego flaga 'sufficient' powoduje, że w przypadku powodzenia stos zakończy się powodzeniem (jeśli wcześniej nie było błędu 'required'). Natomiast w przypadku konta lokalnego, moduł pam_sss.so zgłosi błąd, ale to nie przerywa stosu. Kolejny moduł pam_unix.so wykona się z flagą 'required'. Taki układ powoduje, że zadziała to w przypadku zarówno konta domenowego jak i lokalnego, a przy założeniu, że żaden 'required' nie zgłosi błędu, niezależnie od zalogowanego konta stos może zakończyć się powodzeniem.  
Sugerowane działanie:  
Nie wymagane, plik jest dobrze skonfigurowany a przepływ całego stosu jest prawidłowy i czytelny.

### Przypadek 2

password sufficient pam_unix.so  
password required pam_pwquality.so   
password required pam_pwhistory.so use_authtok   
password sufficient pam_sss.so use_authtok  

Analiza:  
Ten przykład jest skrajnie niebezpieczny, ponieważ pierwszy moduł jest ogromnym błędem zarówno pod kątem flagi jak i umiejscowienia w stosie. Z tego przykładu wynika, że wystarczy podać dowolne hasło, które moduł zaakceptuje lecz nie spełnia polityki haseł, żeby hasło zostało zapisane bez sprawdzenia przez pozostałe moduły. Efekt to słabe lub powtórzone hasło, które być może kiedyś już wyciekło. Choć pozostałe moduły mają poprawne flagi, to nie mają szansy się wykonać. Absolutnie krytyczna i niezalecana konfiguracja.  
Sugerowane działanie:  
- zmienić flagę modułu pam_unix.so na 'requisite' lub 'required', 
- przenieść moduł pam_unix.so na koniec stosu, 
- dodać do modułu pam_unix.so opcję use_authtok.

### Przypadek 3

password required pam_pwquality.so retry=3  
password required pam_pwhistory.so remember=5 use_authtok  
password required pam_unix.so use_authtok  
password sufficient pam_sss.so use_authtok  

Analiza:  
Gdyby nie jeden bardzo subtelny błąd byłby to ten przypadek mógłby przedstawiać poprawną konfigurację typu. Wszystkie moduły mają poprawnie ustawione flagi, wszystkie posiadają wymagane i krytyczne opcje, Jednak kolejność pozostałych dwóch modułów nie jest poprawna. To mało eksponowany na pierwszy rzut oka błąd który można przeoczyć, a który zaburza przepływ typu. Jak to działa: Użytkownik jest zalogowany na koncie domenowym i dla niego zmienia hasło, ale w tym przykładzie najpierw wykonuje się moduł pam_unix.so z flagą 'required'. Moduł ten nie jest przygotowany do kont domenowych i może generować logi typu 'PAM_USER_UNKNOWN' - co może z kolei utrudnić audyt SOC. Problem ten nie występuje dla kont lokalnych. Gdyby były poprawnie ustawione moduły, to przepływ typu był by prawidłowy zarówno dla kont domenowych jak i lokalnych. Przy prawidłowym ustawieniu dwa ostatnie moduły są w odwrotnej kolejności, a dzięki temu pierwszy wykonuje się pam_sss.so i jeśli się powiedzie, stos zakończy się bez błędu (jeśli brak wcześniejszego błędu w 'required'). Jeśli zmiana dotyczy konta lokalnego do moduł ten zwróci błąd, ale to nie zmieni ostatecznego wyniku stosu, który teraz zależy od modułu pam_unix.so z flagą 'required'. To ważne żeby zaznaczyć, że błąd w tym przykładzie nie polega na tym, że hasło do któregoś konta się nie zmieni, ale że w logach będzie błąd, który może utrudnić analizę incydentu.  
Sugerowane działanie:  
- Zamienić miejscami moduły pam.unix.so i pam_sss.so.

### Przypadek 4

password required pam_pwquality.so   
password sufficient pam_sss.so  
password required pam_pwhistory.so use_authtok  
password required pam_unix.so use_authtok  

Analiza:
To kolejna, niezalecana konfiguracja. Wprawdzie flagi modułów są poprawne, jednak kolejność modułów stwarza zagrożenie ponownego użycia hasła podczas zmiany hasła na koncie domenowym. W przypadku gdy moduł pam_sss.so zwróci sukces, to (o ile użytkownik jest zalogowany na koncie domenowym) zostanie zmienione hasło konta domenowego bez sprawdzenia historii haseł - hasło może zostać powtórzone. W przypadku konta lokalnego moduł pam_sss.so zwróci błąd, ale nie przerwie stosu ani nie wpłynie na jego wynik, a moduł pam_pwhistory.so wykona się przed modułem pam_unix.so. Więc dla konta lokalnego wykonają się wszystkie konieczne moduły sprawdzające hasło, lecz dla konta domenowego nie. Ponadto, ponieważ moduł pam_pwquality.so nie ma wpisanych żadnych opcji inline, należy sprawdzić czy plik konfiguracyjny tego modułu (pwquality.conf) ma ustawione konieczne opcje. Podobnie jest z modułem pam_pwhistory.so, w którym nie ma wpisanej opcji inline 'remember', należy sprawdzić czy istnieje plik pwhistory.conf i czy jest poprawnie skonfigurowany. Dodatkowo moduł pam_sss.so nie posiada ustawionej opcji use_authtok, co może stanowić niespójność między modułami, jeśli moduł poprosi o hasło a użytkownik się pomyli.  
Sugerowane działania:  
- zmienić kolejność modułów; moduł pam_sss.so powinien znajdować się przed modułem pam_unix.so i za modułem pam_pwhistory.so,
- dodać opcję inline 'use_authtok' w module pam_sss.so,
- sprawdzić plik pwquality.conf pod kątem konfiguracji opcji,
- sprawdzić czy moduł pam_pwhistory.so ma plik konfiguracyjny - pwhistory.conf w katalogu /etc/security/, a jeśli tak to czy jest tam ustawiona opcja remember. Jeśli nie ma pliku (w niektórych dystrybucjach tak się zdarza) lub w pliku nie ma opcji remember, należy wpisać tę opcję inline w module. 