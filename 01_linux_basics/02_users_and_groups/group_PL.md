# /etc/group

## Cel laboratorium
Celem tego laboratorium było zrozumienie jak działają grupy, jakie zapisy znajdują się w pliku `/etc/group` oraz jak je interpretować. 

## Czym jest /etc/group
Plik `/etc/group` zawiera informacje o grupach w systemie:
- `nazwa grupy` - nazwa grupy systemowej lub użytkownika,
- `hash / marker 'x'` - w większości dystrybucji hasła grupowe są wyłączone i zastąpione 'x',
- `GID (group ID)` - identyfikator grupy - liczba całkowita,
- `członkowie grupy` - lista członków grupy (użytkowników interaktywnych).  
Pole hasła w `/etc/group` jest obecnie nieużywane. Współczesne systemy Linux nie stosują haseł grup, a znak 'x' pełni wyłącznie rolę historycznego wskaźnika kompatybilności. Aktywne hasła grupowe są rzadko spotykane we współczesnych systemach i wymagają weryfikacji. 
Obecnie uwierzytelnianie nie odbywa się za pomocą hasła grupy. Mechanizm PAM uwierzytelnia użytkownika, a nie grupę. Przynależność do grup (np. sudo) jest sprawdzana po pomyślnym uwierzytelnieniu użytkownika jego własnym hasłem. 
Plik nie powinien być edytowany ręcznie, bezpieczna modyfikacja pliku wymaga użycia narzędzi:
- groupadd,
- groupmod,
- groupdel,
- groupmod -aG.
Każda bezpośrednia modyfikacja pliku widoczna w logach, dziwne czasy modyfikacji czy brak wpisu w audit.log to czerwona flaga. Plik powinien posiadać uprawnienia uniemożliwiające zapis wszystkim poza rootem (-rw-r--r--), czyli każdy może czytać, tylko root może zapisywać. Jeśli uprawnienia są rozszerzone to jest sygnał alarmowy, który powinien spowodować natychmiastową kontrolę zmian w pliku oraz przywrócenie odpowiednich uprawnień.

## Plik /etc/gshadow
W starych wersjach systemu plik ten zawierał hashe haseł grup (obecnie nie używa się ich). Zapisy w `/etc/gshadow` mają strukturę:
- `nazwa grupy` - np. sudo,
- `hasło` - hash hasła grupy lub znak blokady (obecnie aktywne hashe w tej grupie mogą świadczyć o potencjalnym niebezpieczeństwie). Oznaczenia '!' i '*' mają takie samo zastosowanie jak w pliku shadow. `!` oznacza zablokowane hasło, `*` oznacza brak hasła - nie można się zalogować.
- `administratorzy` - lista użytkowników, którzy mogą zmieniać członków grupy (używane przez gpasswd - w nowoczesnych systemach Linux ten plik zwykle nie posiada haseł i nie jest używany). Większość grup jest zarządzana centralnie lub ręcznie przez root. 
- `członkowie` - lista użytkowników będących członkami grupy. 
Dostęp do `/etc/gshadow` powinien być ściśle ograniczony (root only), nieprawidłowe uprawnienia to czerwona flaga. 

## Grupy pierwotne (Primary Groups)
Każdy użytkownik ma swoją grupę główną, zapisaną w `/etc/passwd` w polu GID. Ta grupa często nosi taką same nazwę jak użytkownik i jej GID jest taki sam jak UID. Jeśli w grupie jest tylko jeden użytkownik (użytkownik primary, do którego należy grupa) i nie ma członków wtórnych 'secondary', to często w `/etc/group` pole członków tej grupy jest puste, ponieważ jedyny użytkownik wynika z nazwy grupy.
Za pomocą polecenia `ID użytkownik` można sprawdzić rzeczywiste przynależności do grup. Polecenie zwróci GID i nazwy grupy, do których użytkownik należy. 

## Grupy wtórne (Secondary Groups)
To są grupy, które nie powstały podczas dodawania nowego użytkownika, to grupy funkcjonujące osobno i nie mają takiego samego GID jak UID użytkownika. Domyślnie nie należy do nich nikt, dopóki nie zostanie do nich dodany. Prawa by dodać kogoś do grupy ma tylko root lub użytkownik z sudo. Wszystkie te grupy są zapisane w pliku `/etc/group`.

## Dlaczego jest ważne rozróżnienie grup primary od secondary
Grupy primary, w których jest wyłącznie użytkownik, dla którego dana grupa została stworzona jako domyślna, są widoczne zarówno w pliki `/etc/group` jak w pliku /etc/passwd (w passwd wyłącznie GID). W pliku w polu członkowie pojawiają się wyłącznie użytkownicy dodani jako secondary users. W związku z tym samo wyświetlenie grup z pliku `/etc/group` nie pokazuje pełnego członkostwa. 
Należy pamiętać, że plik/katalog tworzony przez osobę w grupie należy do tej grupy. Stanowi to potencjalne zagrożenie przy współdzielonych przestrzeniach, że atakujący, który dostanie się do grupy z podwyższonymi uprawnieniami może w tej przestrzeni tworzyć tworzyć i wyświetlać pliki. To daje możliwość np. podrzucenia złośliwych skryptów. Warto zaznaczyć, że pracując w przestrzeni współdzielonej, należy sprawdzać uprawnienia tworzonych plików lub katalogów. Domyślne uprawnienia zależą od `umask`: pliki -> 666 - `umask` (np. 666 - 022 = 644), katalogi -> 777 - `umask` (np. 777-022 = 755). `umask` działa na zasadzie odejmowania.
Uwagi: sama przynależność do grupy sudo niekoniecznie daje uprawnienia systemowe. Na wielu systemach (np Debian/Ubuntu) uprawnienia dla grup i użytkowników są zapisane w pliku `/etc/sudoers`. 

## Mechanizm kontroli dostępu oparty o grupy
Grupy wpływają bezpośrednio na: 
- `prawa plików` (rwx dla grup),
	- `sticky bit` - działa tylko na katalogu: tylko właściciel pliku, właściciel katalogu lub root może usunąć plik /katalog (drwxrwxrwt),
	- `setgid bit` - na katalogu: nowe pliki dziedziczą grupę katalogu nadrzędnego a nie grupę primary użytkownika (drwxrwxrws), na pliku: podczas wykonywania program działa z GID właściciela pliku a nie z GID użytkownika, który go uruchomił (-rwsr-sr-x). `setgid bit` działa tylko wtedy gdy grupa ma uprawnienia do zapisu w tym katalogu.
- `urządzenia w /dev/`- pliki urządzeń mają owner i group, członkowie grupy mogą korzystać z urządzenia (np. odczyt / zapis). Grupa decyduje kto ma dostęp do sprzętu bez nadawania uprawnień wszystkim (others), 
- `dostęp do logów` - członkowie grupy mają prawo do odczytu logów, inni nie. Pozwala to kontrolować dostęp do wrażliwych informacji tylko przez odpowiednią grupę administracyjną,
- `dostęp do socketów` - członkowie grupy mogą się z nimi łączyć (read/write). Umożliwia to kontrolę, kto może korzystać z serwisów działających na socketach bez nadawania szerokich uprawnień wszystkim,
- `dostęp do docker.sock` (częsty wektor eskalacji) - tylko członkowie grupy docker mogą wykonywać polecenia dockera bez sudo. Pozwala to ograniczyć dostęp do dockera, który daje praktycznie pełne uprawnienia root, tylko do wybranych użytkowników. 

## Współpraca z libc / NSS
Źródłem informacji o grupach nie jest jedynie `/etc/group`, system może używać:
- LDAP,
- Active Directory,
- SSSD,
- NIS.
Plik konfiguracji NSS określający skąd i w jakiej kolejności system ma sprawdzać grupy to `/etc/nsswitch.conf`.
Przykładowa konfiguracja pliku:
```
passwd:		files sss
group: 		files sss
shadow: 	files sss
```
To oznacza:
- że system sprawdzi najpierw lokalne pliki: 
	- `/etc/passwd`,
	- `/etc/group`,
- jeśli nie znajdzie zapyta SSSD - usługi, która komunikuje się z:
	- LDAP,
	- AD,
	- Kerberos,
	- innym katalogiem centralnym. 
Jak to wszystko działa w praktyce:
- Zapytanie:
	- 'id jan'
- System:
	- libc wywołuje NSS,
	- NSS patrzy na nsswitch.conf,
	- dla grup: widzi files sss,
	- szuka:
		- najpierw w `/etc/group`,
		- jeśli nie ma pyta SSSD.
	- SSSD pyta AD/LDAP,
	- dostaje odpowiedź,
	- zwraca wynik jakby był lokalny.
Do sprawdzenia grup w środowisku NSS należy używać polecenia getent, a nie tylko cat `/etc/group`.

## Wnioski bezpieczeństwa
- Jeśli grupa systemowa nie zawiera żadnego użytkownika, jest to normalne i można zignorować,
- Jeśli grupa systemowa zawiera użytkownika interaktywnego, należy sprawdzić, czy powinien mieć w niej dostęp - szczególnie w grupach sudo, adm, docker, 
- Grupy użytkowników interaktywnych należy zawsze dokładnie sprawdzić, aby upewnić się, że nie ma nieautoryzowanych członków. 
	- Oprócz sudo, adm, docker warto sprawdzać grupy takie jak wheel (w niektórych dystrybucjach) i shadow pod kątem nietypowych członków.
	- Każda obecność interaktywnego użytkownika w grupach systemowych wymaga weryfikacji,
- przynależność do grupy sudo nie zawsze oznacza pełne uprawnienia systemowe - by realnie nadać te uprawnienia należy umieścić stosowny zapis w pliku `/etc/sudoers`,
- usunięcie grupy z `/etc/group`. Jeśli grupa zostanie usunięta, ale:
	- użytkownicy nadal mają jej GID jako primary w `/etc/passwd`,
	- pliki w systemie posiadają ten GID jako właściciela grupowego, 
	to:
	- użytkownicy nadal będą mieli ustawiony ten numer GID jako primary lub supplentary w swoich atrybutach procesu,
	- kernel nadal będzie stosował prawa grupowe na podstawie numeru GID,
	- polecenie ls -l może wyświetlać numer zamiast nazwy grupy, 
	- system znajdzie się w stanie niespójności konfiguracyjnej. 
	Usunięcie wpisu grupy nie usuwa jej GID z plików anie procesów. 
- Zmiana GID grupy (bez synchronizacji z `/etc/passwd`). Jeśli zostanie zmieniony numer GID grupy w `/etc/group`, a:
	- nie zostanie zmieniony GID użytkownika w `/etc/passwd`,
	- nie zostaną zmienione właścicielstwa plików (chown / chgrp rekurencyjnie),
	to:
	- użytkownik nadal będzie posiadał stary numer GID,
	- pliki nadal będą posiadały stary numer GID,
	- nowa grupa będzie miała nowy numer, który nie odpowiada istniejącym plikom, 
	- powstanie niespójność między mapowaniem nazwy a rzeczywistymi uprawnieniami.
	System nie dokonuje automatycznej synchronizacji między plikami. 
- Jeśli użytkownikowi zostanie zmieniony primary GID na numer GID grupy sudo:
	- każdy nowo tworzony plik będzie należał do grupy sudo,
	- użytkownik będzie traktowany jako członek grupy sudo w kontekście uprawnień grupowych. Primary GID = membership w tej grupie z punktu widzenia kernela. Więc kernel będzie traktował proces jako należący do tej grupy,
	- może to prowadzić do niezamierzonego rozszerzenia dostępu do współdzielonych zasobów.
	Jednak sama zmiana primary GID nie daje automatycznie uprawnień sudo - te zależą od konfiguracji `/etc/sudoers`. Więc jeśli nastąpiła zmiana primary group GID na GID sudo, a w `/etc/sudoers` jest wpis: %sudo ALL=(ALL:ALL) ALL, to użytkownik dostanie pełne uprawnienia sudo. 
	Uwaga: Jeśli użytkownik NIE jest wpisany w `/etc/group` jako secondary member sudo, to narzędzia typu groups mogą nie pokazywać go jako członka (zależnie od implementacji NSS). Więc analiza samego pliku `/etc/group` jest niewystarczająca, trzeba sprawdzać `/etc/passwd` oraz ID.
- Zasada spójności - Każda zmiana numeru GID grupy primary wymaga:
	- synchronizacji w `/etc/group`,
	- synchronizacji w `/etc/passwd`,
	- ewentualnej aktualizacji właścicielstw plików (`chown` /`chgrp`).
	Brak synchronizacji prowadzi do niespójności i potencjalnych problemów bezpieczeństwa. 
- Duplikacja numeru GID - jeśli dwie różne grupy w `/etc/group` posiadają ten sam numer GID (co normalnie nie powinno być możliwe przy użyciu `groupadd`, ale może wystąpić przy ręcznej edycji pliku):
	- kernel nie rozróżnia nazw grup,
	- procesy należące do jednej z tych grup uzyskają dostęp do zasobów drugiej, 
	- może dojść do niezamierzonego rozszerzenia uprawnień.
	Numer GID jest dla kernela jedynym identyfikatorem grupy, nazwa nie ma znaczenia przy kontroli dostępu. 
- Warto nie przydzielać w pliku `/etc/sudoers` uprawnień zbiorczo dla całej grupy, bezpieczniej tworzyć w katalogu `/etc/sudoers.d/` osobne pliki z indywidualnymi uprawnieniami dla każdego użytkownika - to może ograniczyć eskalację uprawnień przez przypadkowe lub nieuprawnione dodanie kogoś do grupy.

## Analiza wpisów w pliku /etc/group, korelacja z /etc/passwd i /etc/sudoers (Case Study)

### Case study 1
```
Stan systemu:  
/etc/group:   
sudo:x:27:  
/etc/passwd:   
anna:x:1001:27:Anna:/home/anna:/bin/bash  
/etc/sudoers:  
%sudo ALL=(ALL:ALL) ALL  
```
Analiza:  
- Anna jest użytkownikiem, którego primary group to grupa sudo z GID 27, a ponieważ jest użytkownikiem primary tej grupy więc nie widać jej w członkach grupy we wpisie w `/etc/group` (grep sudo `/etc/group` jej nie pokaże),
- procesy użytkownika Anna działają z GID 27, a więc grupy sudo. Kernel nie rozróżnia nazw, widzi jedynie GID, więc procesy Anny mogą dziedziczyć uprawnienia grupy (np. zapis w katalogach grupy sudo). Zachodzi więc ryzyko, że Anna może tworzyć i modyfikować pliki w przestrzeni współdzielonej przez grupę, mimo, że nie widać jej w grupie sudo w `/etc/group`,
- zgodnie z ustawieniami w pliku `/etc/sudoers`, Anna ma jako członek grupy sudo pełne uprawnienia systemowe.   

Sugerowanie działania:  
- należy zweryfikować, czy Anna rzeczywiście powinna być w tej grupie, czy GID jej primary group nie został przypisany celowo lub przypadkowo,
- z perspektywy audytu bezpieczeństwa zaleca się, aby grupa sudo była grupą secondary, ponieważ ułatwia to identyfikację członkostwa w `/etc/group`. Należy więc zmienić grupę primary Anny na własną grupę użytkownika i dodać ją do sudo jako secondary member. Po tej zmianie narzędzi typu groups i ID pokażą poprawne członkostwo, ułatwiając audyt, 
- należy sprawdzić czy i kiedy były edytowane pliki `/etc/passwd`, `/etc/group` i `/etc/sudoers`,
sprawdzić jakie działania w ostatnim czasie Anna przeprowadziła z podwyższonymi uprawnieniami, kierując się jej AUID (Audit User ID).   

Podsumowanie:   
Użytkownik, któremu zostanie przypisana grupa sudo jako primary, nie będzie widoczny wśród członków grupy w pliku `/etc/group`. Będzie miał nie tylko dostęp do plików wspólnych grupy, ale także otrzyma uprawnienia dla grupy skonfigurowane w pliku `/etc/sudoers`.

### Case study 2   
```
Stan systemu:  
/etc/group:  
dev:x:1050:  
backup:x:1050:  
/etc/passwd:  
marek:x:1002:1050:Marek:/home/marek:/bin/bash  
plik:  
-rwxrwx--- 1 root backup script.sh  
```
Analiza:  
- Grupa primary Marka ma GID 1050, gid został zduplikowany i w obecnej sytuacji występują dwie grupy z tym GID. 
- ze względu na zduplikowane GID grup nie da się na podstawie plików `/etc/passwd` i `/etc/group` ustalić, która z nich jest grupą primary Marka, a do której on wcale nie należy,
- marek ma dostęp do plików obu grup, a więc będzie mógł wykonać script.sh. 
- marek ma uprawnienia grupy dev, które obecnie posiada ta grupa w pliku `/etc/sudoers`.  

Sugerowane działania:  
- zweryfikować, która z grup jest pierwotna dla Marka za pomocą:
	- plików historycznych: 
		- `/var/log/auth.log` - mogą być wpisy useradd, usermod, które pokazują pierwotnie przypisane GID,
		- `/var/log/installer` lub dpkg.log w Debian/Ubuntu - informacje o tym, jakie grupy i użytkownicy zostały utworzone podczas instalacji pakietów,
	- jeśli system jest audytowany przez auditd -> ausearch -f `/etc/passwd` lub `/etc/group` pokaże kiedy i jak zmieniono GID,
- zmienić GID grupy primary Marka na niekolidujący z innymi GID za pomocą polecenia:  sudo `groupmod -g nowy_gid dev`,
- sprawdzić grupy z GID 1050, by wiedzieć co ewentualnie zostanie zmienione:  
	`sudo find / -group 1050 -ls` 
- zaktualizować wszystkie pliki należące do starego GID dev za pomocą polecenia:    
	`sudo find / -group 1050 -exec chgrp -h dev {} \;`   
	lub    
	`sudo find / -group 1050 -exec chown :dev {} \;`   
Podsumowanie:  
Konsekwencje tego błędu powodują, że obie zduplikowane grupy mają te same prawa dostępu do plików grupy, ponieważ uprawnienia dla grupy są identyfikowane po GID i jej członkach.  
Sudoers działa po nazwie grupy, nie po numerze GID, dlatego duplikacja GID nie ma wpływu na reguły sudoers. Jeśli w `/etc/group` istnieją dwie grupy z tym samym GID, to zostanie zwrócona pierwsza grupa z listy, w tym przypadku dev (kolejność w `/etc/group` ma znaczenie). Jeśli w `/etc/sudoers` znajduje się wpis %sudo ALL=(ALL:ALL) ALL to oznacza, że użytkownik posiada pełne prawa systemowe. 

### Case study 3
```
Była sobie grupa:  
audit:x:1200:  
Została usunięta z /etc/group, ale:  
w /etc/passwd jeden użytkownik nadal ma GID 1200 jako primary.  
w systemie istnieją pliki:  
-rw-r----- 1 root 1200 raport.log  
```
Analiza:  
- ls -l zobaczy numer GID nie zobaczy nazwy grupy, 
- prawa nadal działają bo kernel widzi numer, który został w passwd, 
- użytkownik będzie mógł odczytać plik raport.log. Jego grupa primary to właśnie grupa audit, bo nie widać użytkownika w liście członków. To rodzaj niespójności konfiguracyjnej.  
- prawa plików nadal będą działać, dla użytkownika, który ma GID 1200 jako primary.  
- nie działają uprawnienia dla grupy określone w pliku `/etc/sudoers`, ponieważ w zapisie ujęta jest nazwa grupy a nie GID. Nie da się więc zidentyfikować nazwy grupy na podstawie `/etc/group`. 

Sugerowane działania:  
- przywrócić wpis w pliku /etc/group w takiej formie jak przed jego usunięciem, za pomocą polecenia:  
	`sudo groupadd -g 1200 audit`   
- sprawdzić własność plików za pomocą polecenia:  
	 `sudo find / -group 1200`  
i jeśli to konieczne polecenia   
	`chgrp audit plik`,
- alternatywnie: zmienić GID w /etc/passwd na istniejącą grupę, ale to wymaga przesunięcia właścicielstw wszystkich plików z GID 1200.  

Podsumowanie problemu:
Grupa przestanie istnieć jako obiekt userspace (brak nazwy w `/etc/group`), ale kernel nadal honoruje numer GID zapisany w pliku `/etc/passwd` i inode plików. Prawa grupowe (rwxrwx---) w plikach są nadal wykorzystywane przez użytkownika, dla którego ta grupa jest grupą primary. Inaczej jest jednak z prawami dla użytkowników secondary tej grupy. Jeśli grupa została usunięta podczas trwania aktywnej sesji użytkownika secondary, to do końca tej sesji są honorowane jego prawa do plików ponieważ:
- podczas logowania:
	- NSS czyta `/etc/group`,
	- buduje listę supplementary GIDs użytkownika,
	- przekazuje ją kernelowi, 
	- kernel zapisuje ją w strukturze procesu. 
Dzięki temu, do końca aktywnej sesji, w której zostaje usunięty zapis w `/etc/group` użytkownik wtórny ma prawa grupowe do plików i katalogów. Traci te prawa po ponownym zalogowaniu, ponieważ nie ma wpisu w `/etc/group`, NSS nie może odczytać go, w supplementary, który stworzy dla kernela, nie będzie tej grupy. Inaczej ma się to trochę do uprawnień systemowych, gdy nie ma zapisu w `/etc/group`, `/etc/sudoers` nie może dopasować reguł opartych o nazwę grupy (bo właśnie po nazwie je przyporządkowuje, a nie po GID), bo grupa przestanie istnieć jako obiekt userspace. 



